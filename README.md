# About Voice Two Factor Authentication
By adding 2FA to your application, you can provide your users effective protection against many security threats that target user passwords and accounts. It will generate a One-Time Password to their phone number via voice call. Application developers can enable two-factor authentication for their users with ease and without making any changes to the already existing application logic or database structure! This guide uses the [Python SignalWire SDK](https://developer.signalwire.com/twiml/reference/client-libraries-and-sdks#python) to show an example of how that can be done!

# Setup Your Environment File

1. Copy from example.env and fill in your values
2. Save new file called .env

Your file should look something like this
```
## This is the full name of your SignalWire Space. e.g.: example.signalwire.com
SIGNALWIRE_SPACE=
# Your Project ID - you can find it on the `API` page in your Dashboard.
SIGNALWIRE_PROJECT=
# Your API token - you can generate one on the `API` page in your Dashboard
SIGNALWIRE_TOKEN=
# The phone number you'll be using for this guide. Must include the `+1` , e.g.: +15551234567
SIGNALWIRE_NUMBER=

```

# Configuring the code 

We need to start by creating variables to store all the data from the authentication session. 

``` python
data = {}
data['requests'] = []
```
Next, we will define a function that looks up the phone number and verifies that the code is correct. We will loop through all authentication sessions, and check if there is a matching number while first adding a + to make sure the number is in E164 format. If there is a session matching that number, we will complete our second check to see if the code they provided was correct. If both conditions are satisfied, we will remove the validated session from the data dictionary created above. 

```python
def lookup_code(number,code):

    # Loop through all sessions
    for i in range(len(data['requests'])):
        # Look if number is equal to a number in a session, we are prepending a '+'
        if '+' + number == data['requests'][i]['number']:
            # Great, We found a session matching that number, now let us check the challenge code
            if code == data['requests'][i]['code']:
                # We have a match, let's remove the validated session and return true
                data['requests'].pop(i)
                return True
    # Catch all for failed challenges
    return False
```

Now that we have defined our variables and our functions, we need to move on to the Flask routes! We will start by defining a route called `/validate-auth` that will validate the authentication request. We will grab the authorization code and the phone number from the GET/POST request and call our previously defined `lookup_code(number, code)` function. If the lookup & authentication are successful, we will return a 200OK. If the lookup is unsuccessful, we will return a 403 FORBIDDEN. 

```python
# Listen for '/validate-auth' route
@app.route('/validate-auth')
def validate_auth():
    # Grab the authorization code from the GET/POST request
    check_code = request.values.get('auth_code')
    # Grab the phone number from the GET/POST request
    number = request.values.get('number')

    # Verify the number and challenge code
    if lookup_code(number, check_code):
        # Return 200, On Accept
        return "200"

    # Return 403, On Forbidden
    return "403"
```
The next route we define is what will be called when you want to create an authentication request. We start by instantiating the SignalWire client using the Project ID, Auth Token, and Space URL. Then we will generate a random 6 digit code between 123456 and 987654. Next, we get the phone number that we're going to send the call to and add both the code and number to our global data object. Lastly, we will use the [Create Call API](https://developer.signalwire.com/twiml/reference/create_a_call) in order to send a call to the requested number with the generated code.

```python
@app.route('/request-auth', methods=['GET', 'POST'])
def request_auth():

    client = signalwire_client(os.environ['SIGNALWIRE_PROJECT'], os.environ['SIGNALWIRE_TOKEN'], signalwire_space_url = os.environ['SIGNALWIRE_SPACE'])

    auth_code = str(random.randint(123456,987654))
    number = "+" + request.values.get('number')

    data['requests'].append({
        'number': number,
        'code': auth_code
    })

    call = client.calls.create(
        from_ = os.environ['SIGNALWIRE_NUMBER'],
          url = "http://" + os.environ['HOSTNAME'] + "/call_2fa?auth_code=" + auth_code,
           to = number
    )

    return "200"
```

Our last route will be used in order to read out the authorization code over the phone call. We will start by getting the auth code from the query params passed in when creating the call in the previous route. We will also instantiate `VoiceResponse()` so that we can use text to speech. Lastly, we will use text to speech to repeat the authorization code twice for the end user and then hang up! 

```python
@app.route('/call_2fa', methods=['GET', 'POST'])
def call_2fa():

    auth_code = request.values.get('auth_code')
    response = VoiceResponse()

    response.say('Your authorization code is: ' + auth_code )
    response.say('Repeating...')
    response.say('Your authorization code is: ' + auth_code )
    response.say('Thank you for calling, GoodBye!')

    return str(response)
```


# Build and Run on Docker

1. Use our pre-built image from Docker Hub 
```

docker pull signalwire/snippets-voice-two-factor-auth:python
```
(or build your own image)

1. Build your image
```
docker build -t snippets-voice-two-factor-auth .
```
2. Run your image
```
docker run --publish 5000:5000 --env-file .env snippets-voice-two-factor-auth
```
3. The application will run on port 5000

# Build and Run Natively

To run the application, execute export FLASK_APP=app.py then run flask run.

You may need to use an SSH tunnel for testing this code if running on your local machine. – we recommend [ngrok](https://ngrok.com/). You can learn more about how to use ngrok [here](https://developer.signalwire.com/apis/docs/how-to-test-webhooks-with-ngrok). 

# Sign Up Here

If you would like to test this example out, you can create a SignalWire account and space [here](https://m.signalwire.com/signups/new?s=1).

Please feel free to reach out to us on our Community Slack or create a Support ticket if you need guidance!

