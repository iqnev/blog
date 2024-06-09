+++
title = 'The impact of the right technology and architecture on project success'
date = 2024-03-23

images = ['images/smart_parking.png']

tags = ['design']

categories = ['Design']

noComment = false
+++

In this post, we will journey through my personal experiences that underscore the critical role
played by apt technology and architecture in project outcomes.
We will see why getting these aspects right from the get-go is of paramount importance.

## Story

I had the opportunity to dive into the fascinating world of parking and traffic systems – a field
ripe with potential for IoT/IIoT solutions.
This was a dream-come-true opportunity for me, reminiscent of my high school days tinkering with
electronics and microcontrollers.

Once I started working on the project, however, I quickly realized that the existing system didn't
meet the business requirement. Delving into the system’s program,
I found it was written in Earlang(it is a language created by the Ericsson Computer Science
Laboratory in the mid-1980s for parallel and distributed computing). Although a renowned
language trusted by top-tier entities like Facebook,
Amazon, and Google, it wasn't the perfect fit for our project,
which necessitated operations on various types of devices like: PT camera, PT VMS, Pay station,
proximity cards (RFID), Automatic License Plate Recognition (ALPR), Near-field Communication (NFC),
Dedicated Short-Range Communications (DSRC), Overhead radars, Bluetooth, LCD panel UI realization
and also
different communication protocols.

My experience with the aforementioned project got me cogitating about a suitable technology that
could help us navigate our way out of the predicament.

Thus, the quest for the ideal technology commenced. It needed to fulfill the following essential
functions:

1. Implement drivers for diverse devices.

2. Implement essential business logic.

3. Implement a user interface for terminal devices, like LCDs. Although Electron had been in use, it
   came with several issues and generally did not seem to be a fitting choice.

In the course of my research, I revisited Qt, a software I had familiarized with during my
university
days. I also had an acquaintance who was proficient in using Qt.

Subsequent to crafting and demonstrating a successful Qt prototype application to the management, we
received approval to transition from Earlang to Qt.

## What is Qt?

To shed more light on Qt, it employs C++ code in conjunction with several non-standard extensions.
Qt is a cross-platform application framework
renowned for developing software applications with graphical user interfaces (GUIs). Notably, Qt
provides tools and libraries to fabricate applications across
desktop, mobile, embedded systems, and web platforms.

A feather in Qt's proverbial cap is its comprehensive feature set. It features a powerful set of
tools for GUI development, cross-platform support, graphics and
multimedia support, and Qt Quick and QML for building modern UIs.

Our decision to switch to Qt was made easier by considering its widespread use globally in popular
platforms like:

1. Desktop
    * KDE Plasma
    * LXQt
    * Unity 2D

2. Embedded and mobile
    * Tesla Model S in-car UI
    * Mercedes
    * webOS
    * MeeGo

3. Platforms
    * Skype
    * Telegram
    * Teamviewer
    * Wireshark
    * VLC media player

[![Qt](/images/qt-platform.png)](https://www.qt.io/)

<br>

## Rethinking Architecture

With Qt in place, we addressed another critical flaw in the project – the underwhelming architecture
unsuitable for a full Machine-to-Machine (M2M) system.

I drafted diagram (**Figure 1**), and want to show you a multi component architecture was
necessitated to
accommodate the solution's requirements and provide a versatile system.
A well-planned multi component architecture would provide a future-ready solution catering to the
customer’s requirements.


<img src="/images/smart_arch.jpg" alt="Smart parking architecture" title="Smart parking architecture">

<br>

_Figure 1 – An example of an IIoT Smart parking architecture_

The graphic(**Figure 1**) above merely depicts the ideal structure of such a system. Regrettably, we
encountered a system devoid of fundamental concepts, prompting us to discard the initial
architecture.


**This issue was exacerbated by the fact that a UI designer served as a crucial advisor to the project!!!**

<img src="/images/kidding.png" alt="Are you kidding me?" title="Are you kidding me?">

<br>

However, we managed to transition our code from Earlang to Qt and introduce a new architectural
paradigm that set the stage for a more robust solution in the future.

## Additional

An IIoT gateway typically consists of the following components:

**Sensors/devices:** These are installed in parking spaces to detect the presence or absence of a
vehicle. Common types of sensors include sensors,actuators, video cameras etc. Field sensors or
actuators are connected to I/O module masters. These I/O module masters transmit data to the
on-premises PLC or IPC. The PLC/IPC is then connected to the IIoT gateway, which serves as a bridge
between the PLC/IPC and the cloud.

**Communication network:** This network carries data between the sensors and the cloud platform.
Common communication protocols include Wi-Fi, cellular, and Bluetooth Low Energy (BLE).

**Cloud platform:** This platform(Parking Guidance System) collects and stores data from the
sensors, as well as manages the overall system. The cloud platform can also provide real-time
information to users about parking availability, as well as allow users to reserve parking spaces.

**User interface (UI):** This is the interface that users interact with to find parking
information and make reservations. The UI can be a mobile app, a website, or a physical display in a
parking lot.

Drawing from the IPSO Alliance, we implemented the IPSO Smart Objects concept. This approach enabled
our system to seamlessly integrate and initialize various
components and create a virtual hardware replica in the Cloud.

## The Triumphant Unveiling

After months of dedicated effort, our team proudly unveiled a fully functional prototype at the
prestigious Traffic Solutions Exhibition in Amsterdam. As we showcased our innovative system to
industry experts and potential clients, the response was overwhelmingly positive. Attendees were
captivated by the seamless integration of cutting-edge technologies and the robust architectural
framework we had meticulously designed.

The prototype's ability to seamlessly orchestrate various components, from automats and cameras to
scanning devices and intuitive user interfaces, garnered widespread acclaim. Visitors marveled at
the system's versatility, real-time monitoring capabilities, and its potential to revolutionize the
parking and traffic management landscape.

The positive feedback we received not only validated our team's hard work but also reinforced our
belief in the power of thoughtful planning and unwavering commitment to excellence. It was a
resounding affirmation that our decision to embrace the right technology and design a future-proof
architecture had paid off handsomely.

As we basked in the success of our triumphant unveiling, we knew that this was merely the beginning
of an exciting journey. Armed with the invaluable lessons learned and the overwhelming support from
industry leaders, we were more motivated than ever to continue pushing boundaries and delivering
innovative solutions that would shape the future of intelligent transportation systems.

## Key Takeaways

Reflecting on these experiences, some key takeaways emerge:

**For Managers**

 * Never entrust critical decisions to people lacking expertise in the relevant area.

 * Selecting the right team members for a project is integral to its success.

**For Engineers**

 * Don't allow your existing knowledge of a programming language or concept to cloud your judgement.
   Remember that each tool is suited for solving specific kinds of problems and
   it's rare for one tool to fit all cases.

 * Never underestimate the power of well-thought-out architecture. It serves as the project's
   foundation and shapes its future evolutions. So always guess that the Voyager ships are still
   running at million km distance from Earth,
   and were designed in the 70s. Engineers then designed a good foundation (architecture), which has
   allowed them to create such well-functioning devices.
   [The Brains of the Voyager Spacecraft: Command, Data, and Attitude Control Computers](https://www.allaboutcircuits.com/news/voyager-mission-anniversary-computers-command-data-attitude-control/)


 * Beware of the Dunning-Kruger Effect, where people with little knowledge often overestimate their
   abilities as they don't comprehend the depth of expertise required to master something.

<img src="/images/dunning-kruger.png" alt="Dunning Kruger" title="Dunning Kruger">

## Conclusion

The journey through this project has taught me invaluable lessons about the significance of making
well-informed decisions and laying a solid architectural foundation from the very start. As
engineers and developers, we often get caught up in the excitement of new technologies or familiar
tools, but it's crucial to step back and critically evaluate whether they truly align with the
project's requirements.

