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
Now we create a script using the command: 
```
nano detect_lang.py
```
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
```
python3 detect_lang.py
```

This will give you the output in the format "<predicted_language> was detected with confidence" for the above texts.

### Analyse sentiment 

### Step 1
Now we create a script using the command: 
```
nano detect_sentiment.py
```
Paste the below script and press CTRL+X and Y and ENTER.

Use the following `detect_sentiment.py` script to test the above texts for sentiment to see if the data is positive, negative or neutral:

```
import boto3

client = boto3.client('comprehend')

# Texts in different languages
Texts = [
    "The French Revolution was a period of social and political upheaval in France and its colonies beginning in 1789 and ending in 1799.",
    "El Quijote es la obra más conocida de Miguel de Cervantes Saavedra. Publicada su primera parte con el título de El ingenioso hidalgo don Quijote de la Mancha a comienzos de 1605, es una de las obras más destacadas de la literatura española y la literatura universal, y una de las más traducidas. En 1615 aparecería la segunda parte del Quijote de Cervantes con el título de El ingenioso caballero don Quijote de la Mancha.",
    "Moi je n'étais rien Et voilà qu'aujourd'hui Je suis le gardien Du sommeil de ses nuits Je l'aime à mourir Vous pouvez détruire Tout ce qu'il vous plaira Elle n'a qu'à ouvrir L'espace de ses bras Pour tout reconstruire Pour tout reconstruire Je l'aime à mourir",
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

    # Detect sentiment using the detected language code
    senti_response = client.detect_sentiment(Text=text, LanguageCode=lang_code)
    sentiment = senti_response['Sentiment']
    sentiment_scores = senti_response['SentimentScore']

    print(f"Sentiment: {sentiment}")

```
Here we just extend from the above code by using the API `client.detect_syntax(Text=text, LanguageCode=lang_code)`
where the lang_code is obtained from the response for the detect_dominant_language(). The response for the detect syntax
is of the format:
{
    'Sentiment': 'POSITIVE'|'NEGATIVE'|'NEUTRAL'|'MIXED',
    'SentimentScore': {
        'Positive': ...,
        'Negative': ...,
        'Neutral': ...,
        'Mixed': ...
    }
}
So, we use the "Sentiment" key of this response to get our sentiment analysis for our text.

### Step2
You can run the script using:
```
python3 detect_sentiment.py
```

This will give you the output in the format "Sentiment: Positive/Negative/Neutral" for the above texts.

### Detect entities

### Step 1
Now we create a script using the command: 
```
nano detect_entities.py
```
Paste the below script and press CTRL+X and Y and ENTER.

Use the following `detect_entities.py` script to test the above texts and find the entities of these texts:

```
import boto3

client = boto3.client('comprehend')

# Texts in different languages
Texts = [
    "The French Revolution was a period of social and political upheaval in France and its colonies beginning in 1789 and ending in 1799.",
    "El Quijote es la obra más conocida de Miguel de Cervantes Saavedra. Publicada su primera parte con el título de El ingenioso hidalgo don Quijote de la Mancha a comienzos de 1605, es una de las obras más destacadas de la literatura española y la literatura universal, y una de las más traducidas. En 1615 aparecería la segunda parte del Quijote de Cervantes con el título de El ingenioso caballero don Quijote de la Mancha.",
    "Moi je n'étais rien Et voilà qu'aujourd'hui Je suis le gardien Du sommeil de ses nuits Je l'aime à mourir Vous pouvez détruire Tout ce qu'il vous plaira Elle n'a qu'à ouvrir L'espace de ses bras Pour tout reconstruire Pour tout reconstruire Je l'aime à mourir",
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

    # Detect entities using the detected language code
    entity_response = client.detect_entities(Text=text, LanguageCode=lang_code)
    for entity in entity_response['Entities']:
        print(f"  - {entity['Text']} : ({entity['Type']})")
    
```
Here we just extend from the above code by using the API `client.detect_entities(Text=text, LanguageCode=lang_code)`
where the lang_code is obtained from the response for the detect_dominant_language(). The response for the detect entity
is of the format:
{
    'Entities': [
        {
            'Score': ...,
            'Type': 'PERSON'|'LOCATION'|'ORGANIZATION'|'COMMERCIAL_ITEM'|'EVENT'|'DATE'|'QUANTITY'|'TITLE'|'OTHER',
            'Text': 'string',
            ....
        }
      ]
    }

So, we use the 'type' and 'text' of the "Entities" response to get the entities and it's type in our texts.

### Step2
You can run the script using:
```
python3 detect_entities.py
```

This will give you the output in the format " 'text' : 'type' " for the above texts.

**What is an entity?**

An **entity** is a specific object that is mentioned in the text. Specifically, an entity can be any name of a person, place, organization or a true date, time, or number.
So basically, entities refer to a true situation or object that has individual meaning when referred to in a sentence, and is marked specifically in the sentence.

### Detect keyphrases

### Step 1
Now we create a script using the command: 
```
nano detect_key_phrases.py
```
Paste the below script and press CTRL+X and Y and ENTER.

Use the following `detect_key_phrases.py` script to test the above texts and get their key phrases:

```
import boto3

client = boto3.client('comprehend')

# Texts in different languages
Texts = [
    "The French Revolution was a period of social and political upheaval in France and its colonies beginning in 1789 and ending in 1799.",
    "El Quijote es la obra más conocida de Miguel de Cervantes Saavedra. Publicada su primera parte con el título de El ingenioso hidalgo don Quijote de la Mancha a comienzos de 1605, es una de las obras más destacadas de la literatura española y la literatura universal, y una de las más traducidas. En 1615 aparecería la segunda parte del Quijote de Cervantes con el título de El ingenioso caballero don Quijote de la Mancha.",
    "Moi je n'étais rien Et voilà qu'aujourd'hui Je suis le gardien Du sommeil de ses nuits Je l'aime à mourir Vous pouvez détruire Tout ce qu'il vous plaira Elle n'a qu'à ouvrir L'espace de ses bras Pour tout reconstruire Pour tout reconstruire Je l'aime à mourir",
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

    # Detect key phrase using the detected language code
    key_phrase_response = client.detect_key_phrases(Text=text, LanguageCode=lang_code)
    key_phrase = key_phrase_response['KeyPhrases'][1]['Text']

    print(f"key_phrases: {key_phrase}")

```

Here we just extend from the above code by using the API `client.detect_key_phrases(Text=text, LanguageCode=lang_code)`
where the lang_code is obtained from the response for the detect_dominant_language(). The response for the detect key phrase
is of the format:
{
    'KeyPhrases': [
        {
            'Score': ...,
            'Text': 'string',
            'BeginOffset': 123,
            'EndOffset': 123
        },
    ]
}
So, we use the 'Text' key in the entity of the 'KeyPhases' key to get the key_phrase of our texts.

### Step2
You can run the script using:
```
python3 detect_key_phrases.py
```

This will give you the output in the format "Key Phrase: key_phrase_of_the_text" for the above texts.

**What is a key phrase?**
A key phrase is any portion of text that adds to the meaning or emphasis of a sentence, even if it isn't unique or descriptive.

### Detect syntaxes

### Step 1
Now we create a script using the command: 
```
nano detect_syntax.py
```
Paste the below script and press CTRL+X and Y and ENTER.

Use the following `detect_syntax.py` script to test the above texts and get the syntax of these texts:

```
import boto3

client = boto3.client('comprehend')

# Texts in different languages
Texts = [
    "The French Revolution was a period of social and political upheaval in France and its colonies beginning in 1789 and ending in 1799.",
    "El Quijote es la obra más conocida de Miguel de Cervantes Saavedra. Publicada su primera parte con el título de El ingenioso hidalgo don Quijote de la Mancha a comienzos de 1605, es una de las obras más destacadas de la literatura española y la literatura universal, y una de las más traducidas. En 1615 aparecería la segunda parte del Quijote de Cervantes con el título de El ingenioso caballero don Quijote de la Mancha.",
    "Moi je n'étais rien Et voilà qu'aujourd'hui Je suis le gardien Du sommeil de ses nuits Je l'aime à mourir Vous pouvez détruire Tout ce qu'il vous plaira Elle n'a qu'à ouvrir L'espace de ses bras Pour tout reconstruire Pour tout reconstruire Je l'aime à mourir",
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

    # Detect syntax using the detected language code
    syntax_response = client.detect_syntax(Text=text, LanguageCode=lang_code)
    text_phrase = syntax_response['SyntaxTokens'][1]['Text']
    text_part_of_speech = syntax_response['SyntaxTokens'][4]['PartOfSpeech']
    text_tag = text_part_of_speech['Tag']

    print(f"{text_phrase} is : {text_tag}")

```

Here we just extend from the above code by using the API `client.detect_syntax(Text=text, LanguageCode=lang_code)`
where the lang_code is obtained from the response for the detect_dominant_language(). The response for the detect syntax
is of the format:

{
    'SyntaxTokens': [
        {
            'TokenId': 123,
            'Text': 'string',
            'BeginOffset': 123,
            'EndOffset': 123,
            'PartOfSpeech': {
                'Tag': 'ADJ'|'ADP'|'ADV'|'AUX'|'CONJ'|'CCONJ'|'DET'|'INTJ'|'NOUN'|'NUM'|'O'|'PART'|'PRON'|'PROPN'|'PUNCT'|'SCONJ'|'SYM'|'VERB',
                'Score': ...
            }
        },
    ]
}

So, we use the text and the "Tag" for the 'PartOfSpeech' to get the syntax from our texts.

### Step2
You can run the script using:
```
python3 detect_syntax.py
```

This will give you the output in the format "Text phrase is: 'Tag'" for the above texts.

**What are syntaxes?**
Syntax displays the order and arrangement of words in a sentence and their level of grammatical roles.

<img width="917" height="465" alt="image" src="https://github.com/user-attachments/assets/f80ade29-fd78-4aa3-a17d-8145fe6c9d05" />

<img width="905" height="442" alt="image" src="https://github.com/user-attachments/assets/dde18be9-70ce-4b0b-95b6-dc5b9d9ebf37" />


## AWS Rekognition

### Add images

You can use the following script `create_lab9_bucket.py` based on thescript in lab 3 to create an s3 bucket and:

Add an image of an urban setting (named as urban.jpg).

Add an image of a person on the beach (named as beach.jpg).

Add an image with people showing their faces (named as faces.jpg).

Add an image with text (named as text.jpg).

into the bucket:

```
import os
import boto3
import botocore

# --- Configuration ---
STUDENT_ID = '23463452'
REGION = 'ap-northeast-1'   
BUCKET_NAME = f"{STUDENT_ID}-lab9"

# --- Initialize S3 client ---
s3 = boto3.client('s3', region_name=REGION)

# --- Create the S3 bucket ---
s3.create_bucket(
            Bucket=BUCKET_NAME,
            CreateBucketConfiguration={'LocationConstraint': REGION}
        )

print(f"Bucket '{BUCKET_NAME}' created successfully in {REGION} region.")

# --- Upload function ---
def upload_file(file_path, file_name):
    """Upload a local file to the S3 bucket."""
    print(f"Uploading {file_name} → s3://{BUCKET_NAME}/{file_name}")
    s3.upload_file(file_path, BUCKET_NAME, file_name)
    print(f"Uploaded: {file_name}")

# --- List of images to upload ---
images = [
    ('urban.jpg', 'Image of an urban setting'),
    ('beach.jpg', 'Image of a person on the beach'),
    ('faces.jpg', 'Image with people showing their faces'),
    ('text.jpg', 'Image with text')
]

# --- Upload each image ---
for filename, description in images:
    if os.path.exists(filename):
        upload_file(filename, filename)
    else:
        print(f"Missing file: {filename} — please place it in the same directory as this script.")

print("\n All uploads complete.")

```
**Note:** Make sure the directories are placed in the same directory as the script.

This code creates an Amazon S3 bucket called 23463452-lab9 in the ap-northeast-1 (Tokyo) region.
First, it initializes an S3 client using boto3, then creates the bucket using the specified region as the location constraint.

Then the program defines upload_file() method which uploads a local file to the S3 bucket using the s3.upload_file() command.
Then it checks to see if there are four specific image files - urban.jpg, beach.jpg, faces.jpg, and text.jpg - in this same folder as the script,
and if it finds any of these files, it uploads these files to the bucket.

If any image file is missing, the program prints out a message to remind the user to place the file in the same folder as the script.
Once all available files are uploaded the program confirms all uploads are complete.

So, first create this script using the command:
```
nano create_lab9_bucket.py
```
Copy paste the above script and the CTRL+X , Y and then ENTER to save it.

Now run it :

```
python3 create_lab9_bucket.py

```

<img width="1366" height="517" alt="image" src="https://github.com/user-attachments/assets/4a0331e3-238d-4a45-973f-16c65b4b53cc" />
<img width="1531" height="621" alt="image" src="https://github.com/user-attachments/assets/72c0421d-1322-45f9-9f6a-75b66161c3af" />


### Test AWS rekognition



