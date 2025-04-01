# Documentation for the Bluetooth Low Energy (Ble) protocol for pretty much any LED Mask controlled by the Shining RGB App

### Characteristics:
- Command Characteristic: `D44BC439-ABFD-45A2-B575-925416129600` (Write)
- Notification Characteristic: `D44BC439-ABFD-45A2-B575-925416129601` (Notify)
- Image Upload Characteristic: `D44BC439-ABFD-45A2-B575-92541612960A` (Write)
- Audio Visualization Characteristic: `D44BC439-ABFD-45A2-B575-92541612960B` (Write)

### Data is encrypted using AES-128 in ECB mode with a fixed key:
`32672f7974ad43451d9c6c894a0e8764`

### Command Structure for the commands that are sent to the command handle:
- 1 byte, length of the command plus arguments
- 1-5 bytes, hex representation of the ASCII command
- x bytes, arguments of the command
- the rest is just padding up to 16 bytes

-----------------------------------------------------------------
## Commands:
### Utility:
- ### `LIGHT`:
  - Description: Sets the brightness, lower brightness means less color accuracy in color depended modes, such as images, and higher brightness over `128` means more flickering due to the LEDs not being able to get that bright at the same flickering frequency. I personally keep it at a max of `100`
  - Hex of the ASCII name: `4c49474854`
  - arguments: 1 byte represents the brightness (0-255)
- ### `IMAG`:
  - Description: Displays the builtin image at the provided id. Everything above `0x69` will display out of bounds data, mostly partial frames of the builtin animations.
  - Hex of the ASCII name: `494d4147`
  - arguments: 1 byte, represents the builtin image id to be displayed (0-255)
- ### `ANIM`:
  - Description: Plays the builtin animation at the provided id. Everything above `0x45` will display out of bounds data, mostly some random pixels. Strangely it still plays that data as an animation
  - Hex of the ASCII name: `414e494d`
  - arguments: 1 byte, represents the builtin animation id to be displayed (0-255)
- ### `DELE`:
  - Description: Deletes the given DIY images from the mask
  - Hex of the ASCII name: `44454c45`
  - arguments: 1 byte gives the count of to be deleted DIY images and the rest give the DIY image ids of the images that should get deleted
 
- ### `PLAY`:
  - Description: Plays the given DIY images in order
  - Hex of the ASCII name: `504C4159`
  - arguments: same as [`DELE`](#dele), but instead of deleting them, it just plays them in order

- ### `CHEC`:
  - Description: Command for checking how many DIY images are on the mask
  - Hex of the ASCII name: `43484543`
  - arguments: triggers a response on the notify Characteristic, sending back the number of DIY images uploaded in one byte

-----------------------------------------------------------------

### Text:
- ### `MODE`:
  - Description: Allows to change the animation used to display the current text
  - Hex of the ASCII name: `4d4f4445`
  - arguments: 1 byte, sets the text display mode:
    - 0: n/a (all n/a are just reverting to off)
    - 1: off
    - 2: blink
    - 3: scroll right to left
    - 4: scroll left to right

- ### `SPEED`:
  - Description: Changes the speed of the text displayment modes
  - Hex of the ASCII name: `5350454544`
  - arguments: 1 byte, sets the speed of the [`MODE`](#mode) effect

- ### `M`:
  - Description: Sets the special color/effect for the text
  - Hex of the ASCII name: `4d`
  - arguments: 1 byte controls wether it's enabled, 1 byte controls the mode:
    - 0: Random color dots on white background
    - 1: Idk, seems like a bit of rainbow on the top and bottom of the text, with red on the left transitioning to purple on the right. A big white part is in the middle
    - 2: Fade from Yellow (top) to Blue (bottom)
    - 3: fade from Green (sides) to blue (middle) (circle shaped)
    - 4: enables the first background image
    - 5: enables the second background image
    - 6: enables the third background image
    - 7: enables the fourth background image
    - everything else: just doesn't change it (aka it just stays at the same option)

- ### `FC`:
  - Description: Sets the text color
  - Hex of the ASCII name: `4643`
  - arguments: 1 byte controls wether it's enabled, 3 bytes control the color in RGB format

- ### `BC`:
  - Description: Sets the background color, can be used to 'disable' the images from the [`M`](#m) command by setting it to black
  - Hex of the ASCII name: `4243`
  - arguments: 1 byte controls wether it's enabled, 3 bytes control the color in RGB format

- ### Text uploading procedure:
  - You will need something that can create a bitmap and, although not required (the text, or part of the text, will just be white if no data is provided for it. For example if you have a bitmap that is `31` pixels wide and color data that has data for only `26` pixels, the last five pixels will be displayed as white), something that can create a color array for this to work. 
  - The bitmap has to be `16` pixels high and could theoretically be a few thousand pixels wide, but I wouldn't recommend doing that as it can cause the mask to become unresponsive and buggy, forcing one to cut the power by removing the battery (for some models, you will have to open the mask to do so). 
    - I would limit the length of the bitmap to about `512` to be save but of course making it `1024` or `2048` could work, it just poses a risk (I also haven't really tested this so I just assumed `512` would be good). 
  - The color array can be used to give each pixel stripe of the text (so 1 x 16, width x height) a different color. 
    - You might need to calculate the total bitmap pixel length and generate an array of bytes with the format RGB `0-f` accordingly. 
    - The special thing about this is that the color stays fixed to the text and not the background like it's the case with the [`M`](#m) command. 
  - Once you have a bitmap, you can use the [`DATS`](#dats) command with the indicator byte set to `0x00` 
    - the first two bytes after the command being the size of the combined data (so bitmap with the color array appended to the end of it, this structure is also used for the uploaded data)
    - the next two bytes being the size of the bitmap alone to initialize a bitmap upload.
  - For the data upload, you just have to send the data to the mask like you would do with the image data, including the [`DATCP`](#datcp) command at the end.

-----------------------------------------------------------------
### Image:
- ### `DATS`:
  - Description: Used to tell the mask that an image upload is about to start
  - Hex of the ASCII name: `44415453`
  - arguments: 2 bytes, image size of the following image, 2 bytes, image index, 1 byte, image toggle (wether to tell that an image is going to be uploaded or a bitmap. Has to be correct, otherwise upload fails)

- ### `DATCP`:
  - Description: seems to confirm the image upload, sending the current unix timecode with it
  - Hex of the ASCII name: `4441544350`
  - arguments: current unix time code

- ### Image uploading process:
  - The program sends the [`DATS`](#dats) command and waits for the [`DATOK`](#datok) answer from the mask. After that, it starts uploading the image.
  - The image is broken up into chunks of `98` bytes per packet (the image size is just the amount of bytes of the image), each byte being either the `red`, `green` or `blue` channel of a pixel. 
    - So for one pixel, three bytes are used. 
    - The first byte (not counting towards those `98` image data bytes) indicates the length of the image bytes in the package, so for the full `98` bytes of image data it would be `0x63`, as there are `98` bytes of image data plus one package counter byte, which starts at `0x00` and goes up by one each package. 
    - After every successfull packet transfer, the mask respons with REOKOK for that image/packet. 
    - The last packet can have less than `98` bytes, so the data length byte is calculated for the less than `98` bytes. 
    - After that, the packet gets padded up to the full `100` bytes.
  - After all packets have been uploaded, the program sends the [`DATCP`](#datcp) command, to wich the mask respons with [`DATCPOK`](#datcpok).

-----------------------------------------------------------------

### Audio:
#### All audio data is sent to the audio visualizer handle, encrypted of course
- #### Audio data:
  - 1 byte for the counter (it's always `0x0f`, as all 15 following bytes are utilized)
  - 1 byte for the mode (modes are listed below)
  - 14 bytes for the visualization. Each nibble of these 14 bytes corresponds to one row of pixels, changing the value of a nibble also changes the size of the pixel row. `0x0` means that that row is off and `0xf` means that that row is 100% full or on.

- #### Modes:
  - 0: vertical bars, color fade from red (middle) over green and blue to purple (left and right sides). The following nibbles are off all the time (starting from nibble 0): 2, 5, 8, 11, 14, 17, 20 (problably also 23 and 26, but those are outside of the screen so they are cut off and never visible in this mode)
  - 1: vertical bars, color fade from red (middle) over green and blue to purple (top and bottom sides). All nibbles are used to control a bars
  - 2: vertical bars, color fade from turquoise (middle) over blue, purple/magenta and red to yellow (left and right sides). The following nibbles are off all the time (starting from nibble 0): 2, 4, 6, 8, 10, 12, 14, 16, 18, 20, 22 (probably also the rest in the pattern, but those are outside of the screen so they are cut off and never visible in this mode)
  - 3: same as 2, but the color fade is reversed and it's not centered in the middle but on the top and bottom sides of the mask.
  - 4: horizontal bars, color fade from light purple (middle) over dark blue, light blue and turquoise to green (top and bottom sides). The following nibbles are off all the time (starting from nibble 0): 2, 5, 8, 11, 14, 17, 20, 23, 26. There are two dots on each side of the screen, on the top and bottom of it. In this mode, if the value of a nibble goes over `0xc`, it just behaves like its value is `0x0`.

-----------------------------------------------------------------
### Responses:
#### All responses are sent on the notification handle and their last byte is always plus `0x01`
- ### `REOKOK`:
  - Description: Confirms that the mask has recieved and processed the image packet
  - Hex of the ASCII name: `52454F4B4F4B`
  - arguments: 2 bytes for the ID of the currently uploading image, 1 byte telling if it's a bitmap or an image (`0x00` if bitmap and 0x01 if image), padding up to 15 bytes, 1 byte for the current packet index with an offset of 1 (it starts to count at `0x01` instead of `0x00`) plus the above mentioned `0x01` (so for packet index `0x50` it would return `0x52`) (I will not mention the addition of that extra byte again, as it's already said above and here)

- ### `DATCPOK`
  - Description: Confirms the recieving of the unix timecode
  - Hex of the ASCII name: `44415443504F4B`
  - arguments: 2 bytes for the two most significant bytes of the unix timecode sent by the [`DATCP`](#datcp) command

- ### `DATOK`
  - Description: Confirms the start of the image upload for the specific image id
  - Hex of the ASCII name: `444154534f4b`
  - arguments: 2 bytes for the image id, 1 byte to tell wether it's an image or a bitmap

- ### `PLAYOK`
  - Description: Confirms that the [`PLAY`](#play) command was executed on the mask
  - Hex of the ASCII name: `504c41594f4b`
  - arguments: none

- ### `DELEOK`
  - Description: Confirms the deletion of images
  - Hex of the ASCII name: `44454c454f4b`
  - arguments: seems to be 32 bytes, I have no idea what those mean. Two [`DELEOK`](#deleok)s are sent for one [`DELE`](#dele) command though

- ### `CHEC`
  - Description: Returns the number of DIY images
  - Hex of the ASCII name: `43484543`
  - arguments: 1 byte for the number of DIY images


-----------------------------------------------------------------

## NOT YET DONE:
- ### `DELEOK`:
  - Data that I know of:
    #### First, here is the data of the [`DELE`](#dele) command:
        0f44454c45140102030405060708090a
    #### and here is the data of the two following [`DELEOK`](#deleok) responses:
        0644454c454f4b02030405060708090b0a0b0c0d0e0f10111213140000000000
        0644454c454f4b111213140000000001

- ### `LOOA`:
  - #### Data that I know of:
    ```
    04 4c 4f 4f 41 b1 91 f8 ec 4b 69 8c 90 1b 2c 37
    04 4c 4f 4f 41 5a 7c b8 3d c5 4e bd dc 97 a0 b7
    04 4c 4f 4f 41 c5 aa 6d 20 43 1c b5 0d b7 bd 92
    04 4c 4f 4f 41 d3 36 d2 49 e5 90 8f 26 8b 94 28
    04 4c 4f 4f 41 65 cd d5 51 db 6b 2f 71 ed 7e bd
    ```

- ### `TIME`:
  - #### Data that I know of:
    ````
    09 54 49 4d 45 00 67 1a 3a fd 00 00 00 00 00 00
    07 54 49 4d 45 45 52 52 3a fd 00 00 00 00 00 01 (this is the answer to the previous command coming from the mask, I'm not entirely sure which byte is still part of the command name and which is data. It could be that the command/response name is TIMEERR)
    ````
