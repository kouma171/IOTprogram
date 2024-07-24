#!/usr/bin/env python3

# Connect the Grove Ultrasonic Ranger to digital port D2

ultrasonic_ranger_D2 = 2
ultrasonic_ranger_D3 = 3

import requests
import json
import time
import threading
import numpy as np
import sys
import cv2
import base64
import grovepi
import time
import math
import random
from grove_rgb_lcd import *

count = 0;
counter = 1;
count_wc = 0;
wc = [0]*25;
wc_re = [0]*25;
wc_count = list(range(5));

def post_image():
    cap = cv2.VideoCapture(0)
    ret, frame = cap.read()
    cv2.imshow('frame',frame)
    path = "photo.jpg"
    cv2.imwrite(path,frame)

    IMAGE = cv2.imread("photo.jpg")
    h,w = IMAGE.shape[:2]
    result = cv2.resize(IMAGE,(int(w/2),int(h/2)))
    cv2.imwrite(path,result)

    _, encimg = cv2.imencode(".png", result)
    img_str = encimg.tostring()
    img_byte = base64.b64encode(img_str).decode("utf-8")
    img_json = json.dumps({'image': img_byte}).encode('utf-8')

    cap.release()
    cv2.destroyAllWindows()

    url = "http://192.168.0.1:5005/get_photo"
    post = {'image':img_byte,"place":"MODE"}
    res = requests.post(url, json = post)
    jsondata = res.json()

def post_wc():
    url = "http://192.168.0.1:5005/get_distance"
    post = {'wc':wc_count}
    res = requests.post(url, json = post)
    jsondata = res.json()

def image_Loop():
    while(1):
        post_image()
        time.sleep(15)

def wc_Loop():
    count = 0
    counter = 0
    count_wc = 0;

    while(1):
        meterD2 = grovepi.ultrasonicRead(ultrasonic_ranger_D2)
        meterD3 = grovepi.ultrasonicRead(ultrasonic_ranger_D3)

        sensor_value1 = 1 if meterD2 < 100 else 0
        sensor_value2 = 1 if meterD3 < 100 else 0
        if sensor_value1 != wc[0] or sensor_value2 != wc[1]:
            count = 2
            while(count < 25):
                wc[count] = random.randint(0,1)
                count = count + 1
            count = 0
            if(meterD2 != 65535 and meterD3 != 65535):
                post_wc()

        if(meterD2 != 65535):
            if(meterD2 < 100):
                wc[0] = 1
            else:
                wc[0] = 0

        if(meterD3 != 65535):
            if(meterD3 < 100):
                wc[1] = 1
            else:
                wc[1] = 0

        while(counter < 6):
            while(count < counter*5):
                if(wc[count] == 1):
                    count_wc = count_wc + 1
                count = count + 1

            wc_count[counter-1] = count_wc
            counter = counter + 1
            count_wc = 0;

        counter = 0
        count = 0
        print(wc)
        time.sleep(0.5)

thread_imageLoop = threading.Thread(target=image_Loop)
thread_wcLoop = threading.Thread(target=wc_Loop)

thread_imageLoop.start()
thread_wcLoop.start()
