---
title: "Waving locks goodbye 👋"
date: 2024-02-13T11:19:47+01:00
tags: [ "python", "ai" ]
---

The code used for this can be found in the following [repository](https://github.com/NikolaMilosa/hand-gestures-lock)

Argh... Where do I even begin... So in the recent weeks I've been stuck with multiple things and I have a buuuunch of things I am working on, reading and investigating... It has gone so bad that I even feel a bit overwhelmed.

So to tackle this I've gone and improved my setup!

## Why?

Just because.

Kidding. It was always fascinating to me to automate simple tasks even if it is just plain typing. To be honest I don't even remember my password for ubuntu login I just know that when I see that screen my hands do the deed and type something out and *voilla* I am in. As for the past couple of months I've been really into the automation with AI and Chatbots and even if it is still not reflected on this blog I am preparing a bunch of things in the playgrounds of my github repositories.

## The problem

What I wanted to achieve is to be able to do simple hand gestures in front of the camera and to be able to login. 

Generally I could've approached this problem in numerous ways but I decided to go with python just because I found a nice already implemented model by Google! (Thanks 🙏) The link to their post is [here](https://mediapipe-studio.webapps.google.com/demo/gesture_recognizer)

I am amazed at the speed of how the AI evolves and how easy is it to set it up and use. If I wanted to build my model I would need to some more things which I didn't need for this. But I do plan on extending the functionallity at some point to use it for controlling different stuff!

## How I did it

Basically I need to do a couple of things:
* Setup camera
* Capture its frames and process them
* Keep track of user input
* End if everything is there

### Setup camera

I really wanted to be fancy here and I've gone ahead and validated the camera. Technically you can really use any camera with this and most laptops have a working one but just in case you have multiple and what to use a different one you can! The code was really simple I just checked with a following function:
```python
def check_camera(camera, logger):
    cap = cv2.VideoCapture(camera)
    if cap is None or not cap.isOpened():
        logger.error("Camera not accessible")
        exit(1)

    return cap
```
This means that you send the camera as a cli argument `--camera` and you can put any integer here as long as it exists as a camera!

### Capture its frames and process them

The biggest issue I have with python is how hard it is to sync between threads normally. If its two threads its okay but if its N threads hoooh... Good luck... Anyway I needed to fetch detect the output in a callback and re-evaluate its response in a main thread... So I used a `Queue`! Here is the general code:
```python
# The callback we used to input the data
def print_result(result: GestureRecognizerResult, output_image: mp.Image, timestamp_ms: int):
    qu.put((output_image.numpy_view(), result.gestures))

while True:
    ret, frame = cap.read()
    
    mp_image = mp.Image(image_format=mp.ImageFormat.SRGB, data=frame)
    recognizer.recognize_async(mp_image, int_timestamp)
    int_timestamp += 1

    frame, gestures = qu.get()

    # Do something with the frame and gestures
```

### Keeping track of user input

First of all we have to tackle how we even setup the combination to check... Since this is something that we need to run at restart of the pc i've gone with cli args with some validation... Here is the code:
```python
def check_combination(combination, logger):
    known = {
        'closed': 'Closed_Fist',
        'open': 'Open_Palm',
        'pointing': 'Pointing_Up',
        'thumbs_down': 'Thumbs_Down',
        'thumbs_up': 'Thumbs_Up',
        'victory': 'Victory',
        'love': 'ILoveYou'
    }

    combination = [elem.strip() for elem in combination.split(',')]
    if len(combination) <= 3:
        logger.error("Combantion too simple. Provide more than 3 simbols")
        exit(1)

    if not all([elem in known for elem in combination]):
        logger.error("Not all tokens are in combination")
        logger.error("Expected keys are: %s", ','.join(known.keys()))
        exit(1)

    return [known[elem] for elem in combination]
```
If you added your own you would have to extend the known combinations.

### Checking if the combination is hit

What a nice way to end it. With some smelly logic! What I did is basically cache the last hit and if the current hit is that don't update it because really what would happen is you spam one guesture even if you didn't change the hand guesture because of FPS!
```python
while True:
    ...

    if len(gestures) > 0 and gestures[0][0].category_name != last_gesture and gestures[0][0].category_name != "None":
        last_gesture = gestures[0][0].category_name
        logger.info(last_gesture)
        attempt.append(last_gesture)
        if len(attempt) == len(to_hit):
            if attempt == to_hit:
                logger.info('Combination hit!')
                break

    # Wait a bit if the combination is missed so we give user enough time to change the guesture.
            logger.warn('Combination missed... Sleeping 3s')
            time.sleep(3)
            attempt = []
            last_gesture = "None"
```

## How does this work together?

All in all the small project turned out well. I love it. Should you use it? Absolutely not. There is number of problems with this. First is that if someone is looking at you they can easily remember the lock. Second is that even if you use these you will probably not be able to print out the things to login screen meaning you will mindlessly be shaping Naruto simbols at the PC... 

Until next time! Cheers!
