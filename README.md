<div style="display: flex; flex-direction: column; justify-content: center; align-items: center; height: 100vh;">

  <h2>Labs 6-9</h2>
  
  <p>Student ID: 23463452</p>
  <p>Student Name: Rishwanth Katherapalle</p>

</div>

# Lab 6

## Set up an EC2 instance

### [1] Create an EC2 micro instance with Ubuntu and SSH into it. 


### [2] Install the Python 3 virtual environment package. 


### [3] Access a directory 


### [4] Set up a virtual environment


### [5] Activate the virtual environment


### [6] Install nginx


### [7] Configure nginx


### [8] Restart nginx


### [9] Access your EC2 instance


## Set up Django inside the created EC2 instance

### [1] Edit the following files (create them if not exist)


### [2] Run the web server again


### [3] Access the EC2 instance


## Set up an ALB

### [1] Create an application load balancer


### [2] Health check


### [3] Access


<div style="page-break-after: always;"></div>

# Lab 7

## Set up an EC2 instance

## Install and configure Fabric

## Use Fabric for automation

<div style="page-break-after: always;"></div>

# Lab 8

## Create a Dockerfile and build a Docker image

## Prepare ECR via Boto3 scripts on your local machine

### ECR

## Push a local Docker image onto ECR

## Deploy your Docker image onto ECS

### Create a task definition for an ECS task:

### Create an ECS service:

### Get a public IP address

## Run Hyperparameter Tuning Jobs


<div style="page-break-after: always;"></div>

# Lab 9

## AWS Comprehend

AWS Comprehend offers different services to analyse text using machine learning. With Comprehend API, you will be able to perform common NLP tasks such as sentiment analysis, or simply detecting the language from the text.

For example, to detect the language used in a given text using boto3 you can use the following code:
```python
import boto3
client = boto3.client('comprehend')

# Detect Entities
response = client.detect_dominant_language(
    Text="The French Revolution was a period of social and political upheaval in France and its colonies beginning in 1789 and ending in 1799.",
)

print(response['Languages'])
```

By executing the code above, we will get something like this:
```
[{'LanguageCode': 'en', 'Score': 0.9961233139038086}]
```
This means that the detected language is 'en' (English) and has a confidence in the prediction greater than 0.99. 

### Detect Languages from different texts

#### [1] Modify the code above

We are modifying the above code to detect different languages using the AWS Comprehend API `detect_dominant_language()`
and `boto3` for texts of 4 different langauges and we format in a way so that the output would be printing message in
the format "<predicted_language> was detected with confidence". Here we replace the language code with it's actual 
name and the confidence is represented as a percentage.

We use these texts from English, Italian, Spanish and French to test the the AWS Comprehend API `detect_dominant_language()`:

**English:**
"The French Revolution was a period of social and political upheaval in France and its colonies beginning in 1789 and ending in 1799."

**Spanish:**
"El Quijote es la obra más conocida de Miguel de Cervantes Saavedra. Publicada su primera parte con el título de El ingenioso hidalgo don Quijote de la Mancha a comienzos de 1605, es una de las obras más destacadas de la literatura española y la literatura universal, y una de las más traducidas. En 1615 aparecería la segunda parte del Quijote de Cervantes con el título de El ingenioso caballero don Quijote de la Mancha."

**French:**
"Moi je n'étais rien Et voilà qu'aujourd'hui Je suis le gardien Du sommeil de ses nuits Je l'aime à mourir Vous pouvez détruire Tout ce qu'il vous plaira Elle n'a qu'à ouvrir L'espace de ses bras Pour tout reconstruire Pour tout reconstruire Je l'aime à mourir"
[From the Song: "Je l'Aime a Mourir" - Francis Cabrel ]

**Italian:**
"L'amor che move il sole e l'altre stelle."
[Quote from "Divine Comedy" - Dante Alighieri]

### Step 1
Now we create a script using the command: ```nano detect_lang.py```.

Paste the below script and press CTRL+X and Y and ENTER.

Use the following `detect_lang.py` script to test the above texts and get our desired output format:

```
import boto3

client = boto3.client('comprehend')

# Texts in different languages
Texts = [
    "The French Revolution was a period of social and political upheaval in France and its colonies beginning in 1789 and ending in 1799.",
    "El Quijote es la obra más conocida de Miguel de Cervantes Saavedra. Publicada su primera parte con el título de El ingenioso hidalgo don Quijote de la Mancha a comienzos de 1605, es una de las obras más
     destacadas de la literatura española y la literatura universal, y una de las más traducidas. En 1615 aparecería la segunda parte del Quijote de Cervantes con el título de El ingenioso caballero don Quijote
     de la Mancha.",
    "Moi je n'étais rien Et voilà qu'aujourd'hui Je suis le gardien Du sommeil de ses nuits Je l'aime à mourir Vous pouvez détruire Tout ce qu'il vous plaira Elle n'a qu'à ouvrir L'espace de ses bras Pour tout
     reconstruire Pour tout reconstruire Je l'aime à mourir",
    "L'amor che move il sole e l'altre stelle."
]

# Dictionary to map language codes to their abbrevations
lang_dict = {
    'en': 'English',
    'es': 'Spanish',
    'fr': 'French',
    'it': 'Italian'
}

for text in Texts:
    response = client.detect_dominant_language(Text=text)
    lang_code = response['Languages'][0]['LanguageCode']
    confidence = response['Languages'][0]['Score'] * 100
    l_name = lang_dict.get(lang_code)
    print(f"{l_name} was detected with {confidence:.1f}% confidence")

```

We use a list name `Text` to store the texts of different languages, which we intend to identify.
Then we map the language codes to the language names to get the output in the desired format using
a dictionary called `lang_dict`.

AWS Comprehend returns two key values for each detected language:
  LanguageCode (like 'en', 'es', 'fr', 'it')
  Score (the confidence value, between 0 and 1) 

To get the desired output, we convert these short codes into full names using this dictionary.
`response = client.detect_dominant_language(Text=text)`.

This line calls the Comprehend API and sends one piece of text at a time to be analyzed in the for loop.
The response is a Python dictionary (JSON-style object) that contains details about the detected languages and confidence scores.

Example response:
`{'Languages': [{'LanguageCode': 'fr', 'Score': 0.9934}]}`.

Now we process this to get the actual name mapped to the language code and the percentage from score by
multiplying it with 100. This gives us our desired output.

### Step2
You can run the script using:

``` python3 detect_lang.py ```

This will give you the output in the format "<predicted_language> was detected with confidence" for the above texts.

### Analyse sentiment 

### Detect entities

### Detect keyphrases

### Detect syntaxes


## AWS Rekognition

### Add images

### Test AWS rekognition

