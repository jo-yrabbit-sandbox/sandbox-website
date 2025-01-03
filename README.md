# Sandbox website

My website: http://sandbox-website-bucket.s3-website.us-east-2.amazonaws.com/
* displays my PGP public key

WIP: Integration with `api-server`
* Prerequisite
  * Make sure `api-server` is running (Does this return string: `http://localhost:5000/api/v1/messages/latest?state=test_state`)
* Host website locally
  * `python -m http.server 8000`
  * Navigate to [http://localhost:8000/](http://localhost:8000/)
* Click `Refresh Data` to
  * display response from [api-server](https://github.com/jo-yrabbit/sandbox-api-server)
  * currently just dumps json content returned by: `http://localhost:5000/api/v1/messages/latest?state=test_state`

## Setup
1. Clone the repository
2. Add PGP key to index.html
3. Deploy to AWS S3

## Things I'm learning
* Display things:
  * Display PGP key
  * --Display images from an image bank in repo--
* Deployed to AWS using Github Action

## Things I want short-term
* Robust way to show content which updates at a higher frequency
  * Prioritize separating dependency between back-/front-end configurations (refresh rates of input/website are independent)
  * Input can be anything at this time, keep it abstract (timestamp/clock that refreshes 1s)
  * Display should take latest snapshot at lower frequency (every 5s) and allow jittering
  * Jittering to prevent race condition betweeen input/display fetching

## Things I want long-pole
* Interact with basic activity in a Telegram group
* Triggers from TG to Website:
  * Display last #postme
  * Tally Y/N responses
* Triggered from Website to TG:
  * Display input text
