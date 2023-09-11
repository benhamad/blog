# Removing bluetooth tethering connection 

I bought a new Laptop last month, an X1 Yoga Gen 6 with 16GB of RAM and an Intel
1185G7 CPU. For a seamless touch screen experience I went ahead and installed
Fedora 38 on it (dual booting with Windows in case I need it).

Playing with it I setup the bluetooth tethering to my phone, which worked though
slowley... But that was only for testing. Now whenever I open the notificaiton
center  -- is that it's name in Gnome -- I see that the Bluetoth tethering is
enabled. There's no option in Gnome to remove it from the UI. So I went ahead
and did it from the command line.

```bash
[chaker@chaker-yoga ]$ bluetoothctl devices
Device 11:22:33:44:55:66 Onyx
Device AA:BB:CC:DD:EE:FF Chaker's phone

[chaker@chaker-yoga ]$ bluetoothctl remove AA:BB:CC:DD:EE:FF
```


final note: I really love the keyboard, I probably wouldn've not be writing this
if not for it. Well, no surprises that I like it coming from a 2019 Macbook
keyboard!

