# Sleep-inator
Final course project for ESE5190

quantumbluee.github.io.

#### Github URL for the project: [Link here](https://github.com/upenn-embedded/final-project-byters.git) 

#### Team Number: 
18    
#### Team Name: 
Byters  
#### Team Members: 
Urvi Haval and Siddharth Ramanathan  

### Demo Video

<iframe width="560" height="315" src="https://drive.google.com/file/d/1h2oaXr2Lgt21I4sSVrcXJzAyCmuN9wzB/view?usp=sharing" 
frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

### Images  
![Poster](Smart_poster.png)
![Components](Components.png)  

### Project Summary  
#### Deivce Description  
- Sleep-inator is a smart pillow to equipped with bluetooth speakers and flexi-sensors. The goal of the project is to help people sleep better and monitor their sleep.  
- The ATMega328 PB acts as the main MCU for this project as it reads the sensor data from flex sensors through the ADC functionality, acts as the power source for the BT speaker system and transmits sensor data to ESP32.

#### Inspiration  
- In today’s increasingly noisy world, finding uninterrupted sleep is harder than ever. From bustling city sounds to unexpected household noises, these disturbances can impact sleep quality and overall well-being.  
- With modern lifestyles often leading to poor sleep quality, the idea was to create a product that not only monitors sleep patterns but also actively aids in improving sleep through adaptive technology.  

#### Device Funtionality 
- The project has 3 main parts: The flex sensor array, the Bluetooth module and the RTC. The code starts the RTC to begin recording the time from 00:00:00, and keeps on updating. The flex sensors are constantly running in parallel, but the values are not read for every ADC trigger. The functionality of the code is to read the sensor values at every 30th minute in order to see if the previous and the current readings are different. In our demonstration, we’ve set the reading duration to be 10 seconds.  
- Once the 10 seconds have elapsed, the sensor values are read from all four sensors in a loop. In each of these individual iterations, the sensor value is read and we compare the difference of this value and the previous reading of this particular sensor 10 seconds ago to be greater than a particular threshold (Threshold value = 10). If this condition is met, we say that there has been a bend in the sensor and this triggers the relay which connects the Bluetooth module. So effectively this brings our main functionality where the bluetooth speakers can play songs or white noise as long as there’s movement detected on the pillow. Here’s the pseudocode for the above explained mechanism:

```c
{
    If 10 seconds have elapsed:
        loop(for each sensor)
    {
            If (current sensor values - previous sensor values)>10
    {
                Relay_on
    }
            Previous sensor value = current sensor value
    }
}
```

#### System Block Diagram  
![Block Diagram](MVP_block_diagram.png)    

#### Firmware Overview

- RTC:
    The real time clock is an I2C device, which led us to write a custom driver. The code structure for the RTC is as follows:
    - The i2c.c file has register and pin configuration commands which were provided in the Atmega328PB’s datasheet. The I2C_Init initialises sets the prescalar to 1 and the SCL frequency to operate in 100kHz. The I2C_Start function sends the start condition of the I2C communication, the I2C_stop send the stop condition, I2C_Write loads the data which is received  as a function parameter, the I2C_Read_ACK will read with acknowledgement and the I2C_Read_NACK will read without acknowledgement. 
    - The rtc.c has functions to Set time and Get time. The RTC_Init function just calls the I2C_Init. The RTC_SetTime function follows an ITC data write operation by calling the I2C functions in the following sequence: I2C_Start(), I2C_Write(address), I2C_Write(data) and I2C_Stop(). The RTC_GetTime function follows an ITC data write operation by calling the I2C functions in the following sequence: I2C_Start(), I2C_Write(address), I2C_Read_ACK(seconds), I2C_Read_ACK(minutes), I2C_Read_NACK(hours) and I2C_Stop(). 

- Main file:
    It begins by configuring the UART for communication, setting up the ADC for analog-to-digital conversion, initializing the relay pin as an output, and preparing the RTC for timekeeping using I2C communication. 
    - The RTC is set to a default time (12:00:00 PM), and the seconds are recorded at every 10-second intervals. 
    - Inside the while loop, the program continually reads the current time from the RTC, converts it from BCD format to decimal format, and formats it as a string to send over UART for logging. 
    - Every 10 seconds, determined by comparing the current seconds with the initial seconds, the program reads sensor values from multiple ADC channels. It compares the current sensor readings to previous values, calculating the difference to detect significant changes which are defined by a threshold.   
    - If a significant change is found in any sensor, the relay is toggled ON; otherwise, it remains OFF. Each sensor value and the current relay status are transmitted via UART for monitoring. 

#### Application logic

1. The RTC module is used to track time and trigger actions every 10 seconds.
2. The flex sensors are connected to the ADC channels and the functionality of a multiplexer is achieved by using ADMUX.
3. At every 10th second, the logic checks if the current value of the sensors is different from the previous value by a threshold. 
4. When the sensors are bent (person is not asleep), the relay toggles and the bluetooth speakers are turned ON. When the sensors are not bent after the next 10 seconds and their value is constant, the relay toggles and the bluetooth speakers are turned OFF.
5. The OFF time of the relay is sent over to the Blynk app using ESP32.


#### Software Requirements Specification & Results

During the coding phase of the project, we followed an iterative development process, progressively building and refining the initial code. 

- **SRS01: Flex sensor array software performance** 
    **Validation:** Initially the plan was to achieve multiplexing using an hardware MUX, but instead the ADMUX register on ATmega328PB was used for its flexibility. Channel switching was implemented successfully by masking the required bits with the respective channel and the ADMUX register. This software requirement also entailed coming up with an appropriate threshold for how different the sensor value is from its previous one. This was achieved by testing various cases by applying pressure on all areas of the pillow. 

```c
void ADC_init() {
    ADMUX = (1 << REFS0);  
    ADCSRA = (1 << ADEN) | (1 << ADPS2) | (1 << ADPS1);  //prescaler of 64
}

uint16_t ADC_read(uint8_t channel) {
    ADMUX = (ADMUX & 0xF0) | (channel & 0x0F); 
    ADCSRA |= (1 << ADSC);  
    while (ADCSRA & (1 << ADSC));  
    return ADC;
}
```

- **SRS02: Implementing the RTC drivers**
    Validation: We chose the DS1307 as the RTC. It was interfaced with the microcontroller via the I2C protocol. The microcontroller's I2C pins were configured with the appropriate pull-up resistors. The driver development consisted of writing the necessary functions according to the configuration required by the driver and the microcontroller. The successful working was verified on the Salae Logic Analyzer. 
    <div align=center>
    <img src="Logic_analyzer.png" width="600"> 
    </div>
    

- **SRS03: Tracking user's sleep time using ESP32 and the Blynk app**
    **Validation:** The project uses the Feather-S2 to transmit the duration for which the relay remains turned off to the Blynk server. This was successfully achieved by establishing communication between the ATmega328PB and the ESP32 and configuring the datastream on the Blynk console. Initially the ESP32 was not able to receive the data from the microcontroller due to some problems in the PC's COM PORT. The cause was verified by using the logic analyzer. The implementation was programmed using the Arduino IDE, ensuring reliable integration and efficient data transmission.
    
    <div align=center>
    <img src="/images/uart_logic.png" width="600"> 
    </div>

    <div align=center>
    <img src="/images/blynkapp.png" width="600"> 
    </div>

#### Hardware Requirements Specification & Results

- **HRS01: Implementing the flex sensor array**
    **Validation:** The flex sensor is not always accurate and did not work the exact same every time. Also, implementing multiple of these in the project was a challenge. Initially, we used the sensors with a voltage divider but upon soldering and attaching them to the pillow we received some junk values. Upon using them with OPAMPs (LM358), the values were significantly consistent. In the current implementation, there are random spikes in the values of the sensor in every 8 or 10 rounds. However, it does not significantly change the working of our project.

- **HRS02: Bluetooth speaker connection and working** 
    **Validation:** The bluetooth speakers are soldered to the bluetooth module and the logic relay. The relay is toggled OFF when there is no change in the value of the flex sensor from the previous value, measured every 10th second. 

**Trickiest Part of the Implementation**

While the sensors work perfectly on the breadboard, when we soldered them and pasted them on the pillow, there seems to be some error in the connections. The flex sensors are giving random values after a few rounds of checking. After a discussion with the TAs, we soldered the flex sensors again.


### 4. Conclusion

The project combined various topics that we learned during ESE5190. We were able to integrate RTC, ADC, UART and drivers for making a really fun project. We also learned how to effectively debug any problem that we might face while working with embedded systems - and the problems are pretty common than what we had thought before.  

Our project idea was fairly straightforward and we did not aim to introduce too complicated concepts right from the start, which helped us build things as we went forward. Speaking of things about the project we are proud of, we were able to implement a software driver from scratch. Using the logic analyzer to see and understand the way I2C works bit by bit was humbling (but fun nonetheless). We did face a few problems with the flex sensors having a mind of their own, but after talking to TAs and researching more about the sensors, we were able to solder them correctly and use a hot-glue gun to make sure there was no contact.  

The next step for the project would be trying to make the pillow as usable as possible, and not just a showcase project. One step we took towards that was sticking the flex sensors on the pillow using command strips so that the pillow can be washed by simply removing the electronics off of the velcro. The Blynk app can also be scaled to provide more analytics regarding the person's sleep over a period of days. The circuitry also needs to be made non-invasive, as currently there is a bunch of wires lying on the side of the pillow. 

Overall, this was a really fun and informative project. All thanks to Nick and the teaching staff (>W<) !

#### Images and Screenshots  
**Blynk App:**  
![Blynk App](blynkapp.png)

**The hero during the MVP presentation:**
![The pillow during the MVP presentation](MVP.jpg)

**Trying to learn i2c :) :**
![Trying to learn i2c :)](n1.png)

**Trying to learn i2c part 2 :) :**
![Trying to learn i2c part 2 :)](n2.png)  

**Trying to learn i2c (you get it now):**
![Trying to learn i2c part 3 :)](n3.png)  

**Showing it off**  
![Showing it off](<WhatsApp Image 2024-12-07 at 11.58.25_dba3ea64.jpg>)  

**Bloopers**  
