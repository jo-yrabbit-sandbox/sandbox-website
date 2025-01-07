# Sandbox website

My website: http://sandbox-website-bucket.s3-website.us-east-2.amazonaws.com/
* Displays my PGP public key
* Lets you download it
* Fetches messages from `api-server`

## How-to run locally for debugging purposes
Assumes your `api-server` is locally hosted at `localhost:5000`

* Host website locally
  * `python -m http.server 8000`
  * Navigate to [http://localhost:8000/](http://localhost:8000/)
* Click `Refresh Data` to
  * Display response from [api-server](https://github.com/jo-yrabbit-sandbox/sandbox-api-server)
  * Currently dumps json content returned by `http://localhost:5000/api/v1/messages/latest?state=test_state`

## Things I want to add/learn/do
Ordered by priority
* Convert this into React (currently html)
  * Take inputs from environment variables, arguments
* Authentication/CORS on server
* Display things:
  * Display text on trigger (user button press)
  * Display images from an image bank within the repo (or from a separate server?)
* Features:
  * Refresh text content at configurable rate
    * Input content updates at a higher frequency than website

### Things I want long-pole
* Respond to chat message exchanges/activity in a Telegram group
* Triggers from TG to Website:
  * Display last stored message
  * Tally Y/N responses (need memory of some sort)
* Can we trigger TG reactions from the website too?
  * Display input text as TG message