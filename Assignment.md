# Assignment: Building a JSON Status API

## I. Overview

Currently, your Pico W REST API returns HTML strings. While this looks good in a browser, it is difficult for other programs (like a mobile app) to read. In this assignment, you will implement a **machine-readable** endpoint that returns the Pico’s status as a **JSON object**.

## II. The Task

Modify `web_server.py` and `main.py` to add a new route: **`GET /status`**.

### 1. Functional Requirements

* **Route:** `GET /status`
* **Data:** The response must include the current temperature and the current state of the LED.
* **Format:** The response body must be a valid JSON string.
* **Header:** The HTTP response header `Content-Type` must be `application/json`.

### 2. Expected JSON Structure

The output in Postman should look exactly like this:

```json
{
  "temperature": 24.55,
  "led_on": true
}

```

---

## III. Implementation Guide

### A. Tracking State

In your `main.py` loop, you are already setting the LED value. However, `web_server.py` doesn't "know" if the LED is currently on or off when a GET request comes in.

* **Tip:** Modify `serve_client(client_socket, temp_c)` to include a third argument: `led_is_active`.
* Pass the current state of the LED from `main.py` into the server function.

### B. Creating the JSON Response

Inside `web_server.py`, you will need to:

1. Import the `json` module at the top of the file.
2. Create a Python dictionary containing your data.
3. Use `json.dumps(your_dict)` to convert it to a string.
4. Call `http_response()` with the `content_type="application/json"` argument.

---

## IV. Submission & Demo

You must demonstrate your working API to the instructor using **Postman**.

### The Demo Steps:

1. **POST** to `/led` with `{"state": 1}`. Verify the physical LED turns **ON**.
2. **GET** `/status`.
* *Check:* Is the status `200 OK`?
* *Check:* Is the body valid JSON?
* *Check:* Does `"led_on"` show `true`?


3. **POST** to `/led` with `{"state": 0}`. Verify the physical LED turns **OFF**.
4. **GET** `/status` again.
* *Check:* Does `"led_on"` now show `false`?



---

## V. Grading Rubric

| Criteria | Points |
| --- | --- |
| **New Route** | `GET /status` is accessible and doesn't crash the server. |
| **Correct Content-Type** | Response Header shows `application/json`. |
| **Valid JSON Body** | The body is a properly formatted JSON object (not HTML). |
| **Dynamic Data** | The JSON accurately reflects the *actual* state of the LED and Temp. |

---
