Here’s a detailed breakdown of the code, step by step:

---

### **Imports**
1. **`from flask import Flask, render_template, request, redirect, url_for`**
   - Used to create and manage the Flask web application. Provides functions to define routes, handle HTTP requests, and render HTML templates.
2. **`from textblob import TextBlob`**
   - A library for processing textual data. It’s used here to perform sentiment analysis.
3. **`from googleapiclient.discovery import build`**
   - Used to interact with Google APIs, in this case, the YouTube Data API.
4. **`import nltk`**
   - Natural Language Toolkit, used for natural language processing tasks.
5. **`from nltk.sentiment.vader import SentimentIntensityAnalyzer`**
   - A sentiment analysis tool that calculates sentiment scores for a given text.
6. **`import pandas as pd`**
   - Pandas is used for handling and analyzing datasets, specifically the Sentiment140 dataset in this script.
7. **`from googleapiclient.errors import HttpError`**
   - Handles errors from the YouTube Data API.
8. **`from langdetect import detect`**
   - Detects the language of a given text.

---

### **Global Setup**
1. **`app = Flask(__name__)`**
   - Initializes the Flask application.
2. **`API_KEY = "Your_api_key"`**
   - Placeholder for the YouTube API key. Replace `"Your_api_key"` with your actual key.
3. **`nltk.download('vader_lexicon')`**
   - Downloads resources needed for VADER sentiment analysis.
4. **Load Sentiment140 Dataset**
   ```python
   sentiment140_df = pd.read_csv('sentiment140.csv', encoding='latin-1', header=None)
   sentiment140_df.columns = ['polarity', 'id', 'date', 'query', 'user', 'text']
   ```
   - Loads the Sentiment140 dataset and assigns column names. This dataset contains tweets labeled with sentiment polarity.

5. **`sia = SentimentIntensityAnalyzer()`**
   - Initializes the VADER sentiment analyzer.

---

### **Routes**
1. **`@app.route("/", methods=["GET", "POST"])`**
   - Defines the root route (`"/"`) which handles both GET and POST requests.
   - **`request.method == "POST"`**:
     - Extracts the video URL from the form data and redirects to the `/sentiment` route with the video URL as a parameter.
   - **`render_template("index.html")`**:
     - Renders the `index.html` page for GET requests.

2. **`@app.route("/sentiment", methods=["GET", "POST"])`**
   - Handles sentiment analysis.
   - Extracts the video URL from the query string, performs sentiment analysis, and renders the results on `home.html`.
   - If the URL is invalid, renders an `invalid_url.html` page.

---

### **Helper Classes**
1. **`class InvalidVideoUrlError(Exception):`**
   - A custom exception to handle invalid YouTube video URLs.

---

### **Sentiment Analysis Functions**
1. **`perform_sentiment_analysis(video_url)`**
   - Extracts the video ID from the URL.
   - Fetches YouTube comments using `get_youtube_comments`.
   - Processes each comment:
     - Detects its language using `langdetect`.
     - Determines sentiment using:
       - Sentiment140 dataset (`analyze_sentiment_from_dataset`).
       - TextBlob or VADER as fallback (`analyze_sentiment_using_textblob_vader`).
     - Categorizes comments as positive, negative, neutral, or unidentified.
   - Returns the sentiment analysis results.

2. **`analyze_sentiment_from_dataset(comment)`**
   - Checks if a comment exists in the Sentiment140 dataset and retrieves its polarity. Returns `None` if not found.

3. **`analyze_sentiment_using_textblob_vader(comment)`**
   - Combines TextBlob and VADER for sentiment analysis.
   - Returns the sentiment score with the highest magnitude (most confident score).

4. **`extract_video_id(video_url)`**
   - Extracts the YouTube video ID from a URL.
   - Handles both standard (`youtube.com`) and short (`youtu.be`) URLs.

5. **`get_youtube_comments(video_id)`**
   - Fetches comments from a YouTube video using the YouTube Data API.
   - Returns a list of top-level comments.

---

### **Error Handling**
- **`InvalidVideoUrlError`**
  - Raised for invalid YouTube URLs.

- **`HttpError`**
  - Catches API errors when fetching YouTube comments.

---

### **Running the App**
- **`if __name__ == "__main__": app.run(debug=True)`**
  - Starts the Flask development server in debug mode for easier debugging.

---

Here's the detailed explanation of every function used in the code, along with the code for each function and a sample **README.md** file for the project.

---

## **Function Explanations**

### 1. **`extract_video_id(video_url)`**
Extracts the YouTube video ID from a given video URL.

```python
def extract_video_id(video_url):
    video_id = None
    if "youtube.com" in video_url:
        video_id = video_url.split("v=")[1]
    elif "youtu.be" in video_url:
        video_id = video_url.split("/")[-1]
    return video_id
```

- **Purpose**: Identify the unique video ID from both standard (`youtube.com/watch?v=...`) and short (`youtu.be/...`) YouTube URLs.
- **Returns**: Video ID as a string or `None` if not found.

---

### 2. **`get_youtube_comments(video_id)`**
Fetches top-level comments for a given YouTube video ID using the YouTube Data API.

```python
def get_youtube_comments(video_id):
    youtube = build("youtube", "v3", developerKey=API_KEY)
    request = youtube.commentThreads().list(
        part="snippet",
        videoId=video_id,
        textFormat="plainText",
        maxResults=350
    )
    try:
        response = request.execute()
        comments = [item["snippet"]["topLevelComment"]["snippet"]["textDisplay"] for item in response["items"]]
        return comments
    except HttpError as e:
        raise InvalidVideoUrlError("Invalid YouTube video URL")
```

- **Purpose**: Uses the YouTube API to fetch up to 350 comments for a video.
- **Returns**: A list of plain-text comments.
- **Handles Errors**: Raises `InvalidVideoUrlError` if the request fails.

---

### 3. **`perform_sentiment_analysis(video_url)`**
Analyzes the sentiment of comments on a given YouTube video URL.

```python
def perform_sentiment_analysis(video_url):
    video_id = extract_video_id(video_url)
    if not video_id:
        raise InvalidVideoUrlError("Invalid YouTube video URL")

    comments = get_youtube_comments(video_id)

    comment_data = {
        "comments_with_sentiment": [],
        "positive": 0,
        "negative": 0,
        "neutral": 0,
        "unidentified": 0
    }

    for comment in comments:
        try:
            language = detect(comment)
        except:
            language = "unidentified"

        if language != 'en':
            comment_data["unidentified"] += 1
            comment_data["comments_with_sentiment"].append((comment, "unidentified"))
            continue

        sentiment = analyze_sentiment_from_dataset(comment)
        if sentiment is None:
            sentiment = analyze_sentiment_using_textblob_vader(comment)

        sentiment_category = "unidentified"
        if sentiment > 0:
            sentiment_category = "Positive"
        elif sentiment < 0:
            sentiment_category = "Negative"
        elif sentiment == 0:
            sentiment_category = "Neutral"

        comment_data["comments_with_sentiment"].append((comment, sentiment_category))
        comment_data[sentiment_category.lower()] += 1

    return "Sentiment Analysis Result", comment_data
```

- **Purpose**: 
  - Extracts video ID and fetches comments.
  - Determines sentiment using datasets and tools.
  - Categorizes comments into positive, negative, neutral, or unidentified.
- **Returns**: Sentiment analysis result string and a dictionary with categorized comment data.

---

### 4. **`analyze_sentiment_from_dataset(comment)`**
Looks up the sentiment polarity of a comment in the Sentiment140 dataset.

```python
def analyze_sentiment_from_dataset(comment):
    matching_row = sentiment140_df[sentiment140_df['text'] == comment]
    if not matching_row.empty:
        return matching_row.iloc[0]['polarity']
    else:
        return None
```

- **Purpose**: Matches a comment against the Sentiment140 dataset and retrieves its polarity.
- **Returns**: Polarity score or `None` if the comment is not found in the dataset.

---

### 5. **`analyze_sentiment_using_textblob_vader(comment)`**
Performs sentiment analysis using both TextBlob and VADER.

```python
def analyze_sentiment_using_textblob_vader(comment):
    blob = TextBlob(comment)
    sentiment_textblob = blob.sentiment.polarity
    sentiment_vader = sia.polarity_scores(comment)["compound"]
    return sentiment_textblob if abs(sentiment_textblob) > abs(sentiment_vader) else sentiment_vader
```

- **Purpose**: 
  - Analyzes sentiment using TextBlob (which calculates polarity) and VADER (which calculates compound score).
  - Chooses the method with the higher magnitude of confidence.
- **Returns**: A sentiment score.

---

### 6. **Custom Error Class**
```python
class InvalidVideoUrlError(Exception):
    pass
```

- **Purpose**: Custom exception to indicate invalid YouTube video URLs.

---
