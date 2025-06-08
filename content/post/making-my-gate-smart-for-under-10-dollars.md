+++
title = 'Making My Gate Smart for Under 10 Dollars'
date = 2025-06-08T13:02:00+09:30
draft = false
+++ 

I recently moved into a new (to me) house with an electric gate, the kind you operate with a remote like this 

![Image.png](https://res.craft.do/user/full/76a116bb-43be-1833-9fc0-33e34d1a381e/doc/8FC10ED4-9A21-4A67-9BD9-B533CBF02468/E75B97F3-C417-48CD-8651-6D9C23463C7A_2/XV85YLhc3DUfBH7QbxyujrJyMIrbGK3eyPA7V7Lr30Ez/Image.png)

# Why

Now I'm about as forgetful as they come, so the prospect of leaving this key at home, or worse, losing it completely, terrified me. I could, with the best of intentions, commit to leaving it in a single place or velcro sticking it to my car, but I know myself too well, so I know it would only be a matter of time until I had an *oh crap* moment and had to call my partner for help.

This gave me an idea, what if I could somehow control it from my phone using either an app or Siri? 

I found various existing options for this online, all ranging from **$100-$200**, this was simply too expensive for me to justify, so I decided I would just make one myself... how hard could it be!?

# How

The first thing I did was remove the cover from the gate control board to look at what inputs and outputs were available to me, and it looked like this:


![Image.png](https://res.craft.do/user/full/76a116bb-43be-1833-9fc0-33e34d1a381e/doc/8FC10ED4-9A21-4A67-9BD9-B533CBF02468/61644349-28AD-4E1B-89CD-3907E847FDE2_2/myvxBV0K6KuacYri23H1qkzcFxJdoUjBrx2TKsi7IK8z/Image.png)

I was particularly interested in that **TRG** pin. So I took a small piece of wire and shorted the TRG and COM pins, and the gate began to swing open! Then, once the gate was closed, I shorted the pins once more and the gate began to swing shut.

Perfect! So if I could somehow remotely trigger the **TRG** (trigger) pin, I could control this gate. The other pin I was interested in was the **LED** pin, this would let me read the current state of the gate. This would be particularly useful for the times when I did still use the remote, as the phyiscal and digital state of the gate would never be out of sync. 

So how to remotely trigger it? Well I'd heard of a thing called an [ESP32](https://www.espressif.com/en/products/socs/esp32), and even though I had no experience with one I knew it was likely the solution to this problem. If you're not familiar with it, it's a tiny microcontroller with Wi-Fi and Bluetooth. 

![Image.png](https://res.craft.do/user/full/76a116bb-43be-1833-9fc0-33e34d1a381e/doc/8FC10ED4-9A21-4A67-9BD9-B533CBF02468/DDF9BF59-1F0D-40A2-B644-86A233A55D29_2/hyBoFNpUizxBYp9QkrEBi7JMCh35NGqvV7PuAC9hMs8z/Image.png)

So I bought an ESP32 development board from Aliexpress for ~$2, a couple GPIO cables, and a 12v to 5V DC-DC converter.

Then I wired it up like this:

![Frame.png](https://res.craft.do/user/full/76a116bb-43be-1833-9fc0-33e34d1a381e/doc/8FC10ED4-9A21-4A67-9BD9-B533CBF02468/27C48215-C65C-414D-94CF-F2E32603D76F_2/yoWXd2kgrqEXmAVfmgYnNkZIZkXcHxlxbEY5U0h2cRsz/Frame.png)

The ESP32 receives power from the auxillary power provided by the gate, but not before being stepped down from 12v to 5v.

Then the trigger pin is wired to GPIO16 and the LED to GPIO4.

# Tying it all together

So now we have a small Wi-Fi enabled board connected to the trigger and LED pins. 

I wanted the eventual control interface for this to be through Apple HomeKit, so I could control it with my voice but also with automations

So to get there, I used a platform called [ESPHome](https://esphome.io), an add-on for [Home Assistant](https://www.home-assistant.io) built to make connecting an ESP32 to the rest of your smart home relatively painless.

## The Code

ESPHome requires you to define everything in YAML, which is a little cumbersome to learn but once you do it's not so bad. The first step is provide a couple setup options 

```other
esphome:
  name: learning
  friendly_name: Gate
  on_loop:
    then:
      - lambda: |-
          static bool last = false;
          static uint32_t last_change = 0;
          bool current = id(gate_led_raw).state;
          uint32_t now = millis();
          if (current != last) {
            ESP_LOGD("gate_led", "State changed to %d after %d ms", current, now - last_change);
            last = current;
            last_change = now;
          }
esp32:
  board: esp32dev
  framework:
    type: arduino
```

Next we setup our inputs and outputs:

```other
binary_sensor:
  - platform: gpio
    pin:
      number: 4
      mode: INPUT
    name: "Gate LED Raw"
    id: gate_led_raw
    internal: true
    filters:
      - delayed_on: 50ms
      - delayed_off: 50ms

output:
  - platform: gpio
    pin: 16
    id: gate_trigger
    inverted: false

button:
  - platform: output
    output: gate_trigger
    duration: 500ms
    name: "Trigger Gate"
    id: gate_button
```

A very important part of this is the `filters` section above. As LEDs use PWM, initially the gate kept reporting it was open, then closed, then open, then closed, multiple times a second. This filter ensures the LED must be either ON or OFF for at least 50ms before my code proceses things.

The final step is to expose a *cover* from ESPHome to Home Assistant. A cover is a device class which encompasses all number things such as smart blinds, curtains etc. It's also the kind of device class we use to expose our gate, as while we care primarily about open and closed, it can also exist at any point in between.

```other
cover:
  - platform: template
    name: "Driveway Gate"
    id: gate_state
    device_class: garage
    open_action:
    - if:
        condition:
          lambda: 'return id(gate_state).position < 1.0;'
        then:
          - logger.log: "Triggering button to open."
          - button.press: gate_button
    close_action:
    - lambda: |-
        ESP_LOGD("main", "Asked to close, current position is %.2f", id(gate_state).position);
    - if:
        condition:
          lambda: 'return id(gate_state).position > 0.0;'
        then:
          - logger.log: "Triggering button to close."
          - button.press: gate_button
    stop_action:
      - logger.log: "Asked to stop, triggering button."
      - button.press: gate_button
```

Now if the gate is asked to open, close or stop, it will send `HIGH` to the `TRG` pin on the smart gate for 500ms. It first checks the current state, to ensure if we ask to open when it's already open, it doesn't inadvertently close it.

Tracking the state was the hardest bit to get right, it took a couple weeks of trial and error before things settled in, but here is where I landed:

```other
static uint32_t last_change = 0;
static bool last_state = false;
static uint32_t stable_time = 0;
static bool stable_state = false;
const uint32_t STABLE_THRESHOLD = 2000;  // ms

bool current = id(gate_led_raw).state;
uint32_t now = millis();

if (current != last_state) {
  last_change = now;
  last_state = current;
}

if ((now - last_change) > STABLE_THRESHOLD && current != stable_state) {
  stable_state = current;
  stable_time = now;
}

if ((now - stable_time) < 3000) {
  return COVER_OPERATION_OPENING;  // or CLOSING, hard to tell
}

return stable_state ? COVER_OPEN : COVER_CLOSED;
```

It checks the state of the LED as well as when it last changed. Once it has been stable for 2s we publish either COVER_OPEN or COVER_CLOSED. This way if my partner uses the clicker to open the gate, the Home app on my iPhone will also show the gate as either open or closed.

And just like that, with a little extra tinkering in the Home Assistant side, I could control the gate from my iPhone

![Image.png](https://res.craft.do/user/full/76a116bb-43be-1833-9fc0-33e34d1a381e/doc/8FC10ED4-9A21-4A67-9BD9-B533CBF02468/38A74CD1-6FE8-4428-92D3-8F61D6403ECA_2/228ZDWC6AaflhBWrZQfyT8bVWHFr1sNZvaiS0VV36Hwz/Image.png)


This was my first ESP32 project, and I'm so thrilled with how it turned out. When I arrive home, my gate automatically opens, and when I get in my car at home it automatically opens too.

I'm sure I made mistakes, so if you spot anything I did wrong or could do better please let me know!