# Send data to AttnGAN from Arduino Uno

The goal will be to use a potentiometer to navigate through the lines of a poetry and generate an image for each line using the AttnGAN model.

## List of parts necessary

- an Arduino Uno
- a breadboard
- a 10kohms potentiometer


## RunwayML setup
RunwayML is pretty straight forward :
- In the model directory look for the "AttnGAN" model, which will take up some text to transform it to an image.
- Run it. 
- Click on the network tab.
- Make sure that in the input window HTTP is selected.

The network tab will give useful information on how the input should be formatted eg **Input Specification**, the **server address** and **port** number, as well as the usable **routes** to get/send data from/to the model.

In our case :

- We should send a **JSON** string with this kind of data :

    ```json
    {"caption":"the text I want to translate to image"}
    ```

- We want to send data to RunwayML so we need to use the **POST** route from Arduino to our computer running RunwayML so our httpRoute will look something like :

    ```
    http://my-computer-ip:runway-port/query
    ```

As everything will happen locally on your computer you don't need to bother to know your IP address. Using 'localhost' should be enough.


## The circuit

Our circuit will be as follows :

![circuit](circuit.png)

## Running the examples

### With p5.js

- First you need to make the connections between the potentiometer and the Arduino card as shown just above.

- Then you need to flash your Arduino board with the code provided as an **.ino** file. If you don't know how to do this you can follow this guide : https://www.arduino.cc/en/Guide/ArduinoUno#toc2

- After that you need to open the **p5.serialcontrol** application, select the same board you upload your program to and click on 'open' :

![p5.serialapp](../assets/p5_serial_capture.png)

This will act as a server and forward the data you write on the serial port to our webpage.

- Launch RunwayML, set it up as mentioned above, and click on 'run'.

- Go to this page : https://editor.p5js.org/b2renger/sketches/HSu5bjj3P
and click on 'play'.

### With Processing

- First you need to make the connections between the potentiometer and the Arduino card as shown just above.

- Then you need to flash your Arduino board with the code provided as a **.ino** file. If you don't know how to do this you can follow this guide : https://www.arduino.cc/en/Guide/ArduinoUno#toc2.

- Launch Processing.

- Install the RunwayML library (if you don't know how to do this you can find more there : https://github.com/processing/processing/wiki/How-to-Install-a-Contributed-Library).

- Launch RunwayML, set it up as mentionned above, and click on 'run'.

- Run the Processing sketch provided.


## Arduino code break down

### Read data from a potentiometer
The Arduino code will read the value from a potentiometer and map it to an index so we can fetch a line of poetry stored in an array of strings.

From this line we will build a JSON object with the specifications mentionned above and simply print it in our serial output.

First things first we will open a serial connection in the setup() function :
```c
void setup() {
  Serial.begin(9600);
}
```
Then we need to read a value on the pin **A0** and [map](https://www.arduino.cc/reference/en/language/functions/math/map/) it to a new value (our poetry will have 5 lines).

```c
void loop() {
  int index = map(analogRead(0), 0, 1023, 0, 5);
}
```

To avoid printing a lot of data and sending a lot of data to our middleware application, we will be sure that data is written to the serial port only when the 'index' value has changed. For this we will create a global variable called *pindex* that will keep track of the last value of *index* which is our current reading. So if the value of *index* is not the same as *pindex* that means something has changed and we can send data to our middleware.

So before the setup() function, you need to create a variable :

```c
int pindex = 0;
```

In the draw() function you need to make a test :

```c
// test if the value has changed from previous frame
if (pindex != index) {
  pindex = index;
  // (...) more code to write data to serial
}
```
### Store and access lines of text
Our data is a line of poetry, and to access this data we will store our poetry in an array called, *String*. We can declare it and initialize it before the setup() function :

```c
// Alfred Lord Tennyson - THE EAGLE
String  poem [] = {
  "He clasps the crag with crooked hands;",
  "Close to the sun in lonely lands,",
  "Ring’d with the azure world, he stands.",
  "The wrinkled sea beneath him crawls;",
  "He watches from his mountain walls,",
  "And like a thunderbolt he falls."
};
```

This means we can access every line easily with an index value, for instance :

```c
Serial.println(poem[1]); // returns "Close to the sun in lonely lands,"
```

### Format to JSON an post to serial bus
Now we just have to format this data to a JSON object and print it to our serial bus. We use the [arduinoJSON](https://arduinojson.org/) library for that - you can follow [this guide](https://www.arduino.cc/en/guide/libraries) to do so.

First we need to include the library in our code. This should be the very first line :
```c
#include <ArduinoJson.h>
````
Then in between the brackets of the *if* statement we wrote earlier we will :

- Create a JSON object
  ```c
  // Create a JSON object with the correct formatting from our poem
  // Use arduinojson.org/assistant to compute the capacity.
  // use the longest phrase of the poem to compute the size
  const size_t capacity = JSON_OBJECT_SIZE(1) +55 ;
  DynamicJsonDocument root(capacity);
  ```

- Add an entry to it
  ```c
  // Add data to it : key == "caption" / content == line from our poem
  root["caption"] = poem[index];
  ```

- Write the data to the serial bus
  ```c
  // write data to a string with JSON formatting
  String json;
  serializeJson(root, json);
  Serial.println(json);
  ```
The data is now ready to be retrieved and processed in p5.js or in Processing.

## Code break down for p5.js

You will need to have the **p5.serialcontrol** application running and connected to your Arduino card, running the Arduino code example provided.

You will be able to find the p5.js client code here : https://editor.p5js.org/b2renger/sketches/HSu5bjj3P

If you start from scratch, note that you will need to add the [p5.serialport](https://github.com/p5-serial/p5.serialport) library to your **index.html** file :

![index.html](../assets/p5js_index.html.png)

Line 7 :
```html
<script language="javascript" type="text/javascript" src="https://cdn.jsdelivr.net/npm/p5.serialserver@0.0.28/lib/p5.serialport.js"></script>
```

### Setting up the serial connection
To get the data from the potentiometer we need to create a variable to deal with our serial connection. This variable will be an instance of a *serial* object on which we can call several functions.

First let's create a variable before the setup() function:

```js
let serial; // instance of a serial connection
```
Then in the setup() function we will configure everything we need :

- Initialise the object :
  ```js
  serial = new p5.SerialPort();
  ```
- Connect it to our physical Arduino board :
  ```js
  // Assuming our Arduino is connected, let's open the connection to it
  // Change this to the name of your Arduino's serial port
  serial.open("/dev/tty.usbmodem145101");
  ```
- Define a callback function to be sure our connection is operational :
  ```js
  // When we connect to the underlying server
    serial.on('connected', serverConnected);
  ```
  This actually means that we need to create a new function that will be called when we are connected to the server which is run by the *p5.serialcontrol* application.
  (This code needs to be outside the *setup()*)
  ```js
  // We are connected and ready to go
  function serverConnected() {
    print("Connected to Server");
  }
  ```
- Define a callback function to write the code we need to execute when data arrives :
  ```js
  // When we some data from the serial port : callback the 'gotData()' function
  serial.on('data', gotData);
  ```
  We will define it and leave it empty for now.  (This code needs to be outside the *setup()* function)
  ```js
  // There is data available to work with from the serial port
  function gotData() {
  }
  ```
Now the connection between our Arduino code and our webpage should be working, and we can now get the data and process it. We will now mainly write code in the *gotData()* function we just defined.

### Receiving data from Arduino
When data is written by Arduino to the serial bus, our JS code automatically fires up the **gotData()** function. To read the data we can use the **serial.readLine()** function.

```js
// There is data available to work with from the serial port
function gotData() {
  let currentString = serial.readLine(); // read the incoming string
  trim(currentString); // remove any trailing whitespace
  if (!currentString) return; // if the string is empty, do no more
  console.log(currentString);
}
```
One thing that is important is to parse the JSON string we receive to be able to use it correctly. We will want to display the received text in our canvas, so we need to create a variable to store it - we do that before the setup() function :
```js
let txt;
```
In the **gotData()** function we will use the **[JSON.parse](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/parse)** function from vanilla javascript to convert our JSON string to a valid JSON object and store the result in our new variable :
```js
  txt = JSON.parse(currentString); // parse the JSON string read from Arduino to a JSON object
  console.log(txt);
```
This way we can display it in our canvas by adding this line of code to the **draw()** function :

```js
 if (txt != null) text(txt.caption,width*0.5, height - 20);
 ```
Thanks to the json.parse() function we are able to access the key inside a JSON object using a dot '**.**'.

Now we want to send our JSON object to RunwayML for it to generate an image.

### Sending data to RunwayML
To do so we will use the **[httpPost()](https://p5js.org/reference/#/p5/httpPost)** function from the p5.js library.

This function will help us to post a request to a HTTP server by specifying the *path*, the *data* and a callback function to handle the *response*.  

The path is given by RunwayML in the network tab (see above in the RunwayML setup section). The data is the JSON object we extracted and not the JSON string received from Arduino.

First we will leave the response callback function empty :

```js
// post the JSON file to RunwayML and handle the response as an anonnymous function
  httpPost("http://localhost:8000/query", txt, function(response) {
    console.log(response)
  })
```

### Receive data back from RunwayML
Now it's time to complete the code of our anonymous callback function that is passed to our httpPost request.

The response of RunwayML will be an image. To handle it we will create a variable to store and display the image. So before the setup() function we create a new variable :

```js
let img; // hold the image generated by RunwayML and received with http
```
And in the callback we will parse the response and load the image in this new variable :

```js
httpPost("http://localhost:8000/query", txt, function(response) {
    //console.log("got response");
    let data = JSON.parse(response);
    img = loadImage(data.result);

  })
```
We can now draw the received image to our canvas by adding this final line of code to the draw function :

```js
if (img != null) image(img, width*0.5, height*0.5, width,height);
```

## Code break down for Processing

As mentioned above, the Processing code uses two libraries :
- the [serial library](https://processing.org/reference/libraries/serial/index.html), included in the Processing dowload.
- the [RunwayML ML Processing Library](https://RunwayML.github.io/processing-library/), which you should download and install via the contribution manager or manually (see [here](https://github.com/processing/processing/wiki/How-to-Install-a-Contributed-Library) if you need help with this part).

### Serial connection
First things first, we need to make our Processing program and our Arduino board to talk to each other. As we already covered the Arduino code, we just need to tell Processing which Arduino board we use and define a function that will retrieve and process data as it was sent by the Arduino.

#### Setting up the serial connection

Before the setup() function we need to first import ou serial library and create a variable that will hold an instance of the port we will connect to :

```java
// import serial libraty
import processing.serial.*;
// instance of the port we will be connecting to
Serial myPort;  
```

In the setup() function this time we will initialise the port to talk with the Arduino board :

```java
printArray(Serial.list()); // list the available serial devices connected
String portName = Serial.list()[3]; // our board is the third in the list (change accordingly)
myPort = new Serial(this, portName, 9600); // this app - 3rd board in the list - talking speed
myPort.bufferUntil('\n'); // read the data line by line
```

#### Receiving and parsing data

The serial library provides a utility function that will fire up when code is received on the serial port, this is the [serialEvent](https://processing.org/reference/libraries/serial/serialEvent_.html) function.

It is defined like that outside the scope of the setup() or the draw() functions : 
```java
// this function triggers itself when a message is receive via serial
void serialEvent (Serial myPort) {
}
```

The example in the documentation is quite minimal so we will do a lot of tests and encapsulate them in a [try-catch]{https://processing.org/reference/try.html} function to avoid crashes if something goes wrong in the communication. (By experience, it helps a lot when working with this kind of connection between Arduino and processing.)

```java
void serialEvent (Serial myPort) {
  try {
    while (myPort.available() > 0) {  // if connection is ok
      String inBuffer = myPort.readStringUntil('\n'); // read line by line
      if (inBuffer != null) { // is buffer is not null
        if (inBuffer.substring(0, 1).equals("{")) { //parse the begining { 
          JSONObject json = parseJSONObject(inBuffer); // parse the JSON string to a JSON object
          if (json != null) { // if JSON object is not null

            // here we can manipulate data and do things !
            
          }
        }
      }
    }
  } 
  catch (Exception e) {
  }
}
```
So this is a lot of code, but right in the middle there is an if statement in which we will be able to code some things.

First we want to get the text sent by Arduino. We need for this to have a variable ready, so before the setup() function we can create a new variable :

```java
// The caption to display 
String txt;
```
Then inside the before mentionned **if** statement, we can fill it with the data we received via Arduino :

```java
// read the data and store it in a variable
txt =json.getString("caption");
```

And finally we can display it, in the draw() function of our program :

```java
background(0); 
if (txt!=null) {
  textAlign(CENTER, BOTTOM);
  textSize(12);
  stroke(255);
  fill(255);
  text(txt, width/2, height);
}
```
Now we need to send this data to RunwayML and handle its response, which will be an image generated by AttnGan.

### Communication with RunwayML
The **Runway ML Processing Library** will pretty much ease everything in the communication with RunwayML; it has a lot of examples available for many use cases. The code described here is heavily adapted from two examples shipped with the library :
- attnGANBasics
- attnGANPrompt

First things first again, we should import the library and create a variable to refer to an instance of RunwayML :
```java
// import RunwayML library
import com.RunwayML.*;
// reference to RunwayML instance
RunwayHTTP runway;
```

In the setup we should now initialise everything :
```java
// setup RunwayML
runway = new RunwayHTTP(this);
// don't send messages to Runway continuously
runway.setAutoUpdate(false);
```
That is all ! The library takes care of ports and address for us !

#### Sending data
To send data we can do that in two lines of code. This code should be placed inside the **serialEvent** function as well as inside the try-catch and inside the **if** statement :

```java
String data = json.toString(); // interpret the JSON object as a string
runway.query(data); // send it to RunwayML
```

Here is the full function - to visualize better :

```java
// this function triggers itself when a message is receive via serial
void serialEvent (Serial myPort) {

  try {
    while (myPort.available() > 0) {  // if connection is ok
      String inBuffer = myPort.readStringUntil('\n'); // read line by line
      if (inBuffer != null) { // is buffer is not null
        if (inBuffer.substring(0, 1).equals("{")) { //parse the begining { 
          JSONObject json = parseJSONObject(inBuffer); // parse the JSON string to a JSON object
          if (json != null) { // if JSON object is not null

            // read the data and store it in a variable
            txt =json.getString("caption");
            // send the data to RunwayML
            String data = json.toString(); // interpret the JSON object as a string
            runway.query(data); // send it to RunwayML
            
          }
        }
      }
    }
  } 
  catch (Exception e) {
  }
}
```

#### Receiving data
To receive data the library provides a **runwayDataEvent** function that will be called each time data from RunwayML is available to process.

```java
// this is called when new RunwayML data is available
void runwayDataEvent(JSONObject runwayData) {
}
```

To get the data from attnGAN we just have to do a :
```java
String base64ImageString = runwayData.getString("result");
``` 

This will get us an image formatted as a base64 string, that can't be read directly by Processing. Luckily the example *attnGANPrompt* shows how to decode it and store it in a variable. We will need to create a variable of the type PImage to store the result and the display it.

So before the setup we create such a variable :

```java
// result image will be stored here
PImage runwayResult;
```
And inside the runwayDataEvent function we will call the utility functions to try and decode the base64 image we just received - again using a try-catch to avoid eventual crashes
```java
try {
    PImage result = ModelUtils.fromBase64(base64ImageString);
    if (result != null) {
      runwayResult = result;
    }
}
catch(Exception e) {
  e.printStackTrace();
}
```

We can now display the decoded image in the main draw() function :

```java
if (runwayResult != null) {
    image(runwayResult, 0, 0, width, height);
}
```









