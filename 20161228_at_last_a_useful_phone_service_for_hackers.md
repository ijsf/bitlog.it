# At last, a useful phone service for hackers!

_2016/12/28 Oguz Meteer // guztech_

---

Imagine the following, all too often occurring scenario: you are in the supermarket, and suddenly you feel the urge to convert the price of that loaf of bread to hexadecimal. But how are you going to do that with just your dumbphone?

Easy!

It all started with finding a service that offers a cheap SIP service and an incoming landline. Now, the price might be right, but what are you going to use it for, other than the obvious? Well, why not create a phone service with a Python script?

Our code imports the Python bindings of the _pjsip_ library, which provides basic SIP client functionality as well as sound playing functionality. The script uses the pjsip library to register itself as a SIP client to our VoIP provider's SIP service. The general idea is to handle incoming calls, capture any DTMF digits (keypad digits), do some conversion logic, play the appropriate wave files and hang up again.

The Python script at the heart of this service is available at [github.com/GuzTech/hackers_conversion_service_sip](https://github.com/GuzTech/hackers_conversion_service_sip).

Happy hacking!
