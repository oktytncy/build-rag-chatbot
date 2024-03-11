
## 1️⃣ Getting started with Streamlit to build an app

Let us now build a real application we will use the following architecture

![steps](/./assets/steps.png)

In this workshop we'll use Streamlit which is an amazingly simple to use framework to create front-end web applications.

To get started, let's create a hello world application as follows:

```python
import streamlit as st

# Draw a title and some markdown
st.title("Your personal Efficiency Booster")
st.markdown("""Generative AI is considered to bring the next Industrial Revolution.  
Why? Studies show a **37% efficiency boost** in day to day work activities!""")
```
The first step is to import the streamlit package. Then we call `st.title` to write a title to the web page and lastly we write some markdown content to the web page using `st.markdown`.

Check out the complete code in [app_1.py](./app_1.py).

**If you want to test and run the app:**
```bash
streamlit run step-by-step-app/app_1.py
```

## 2️⃣ Add a Chatbot interface to the app

In this step we'll start preparing the app to allow for chatbot interaction with a user. 

**We'll use the following Streamlit components:**
1. `st.chat_input` in order for a user to allow to enter a question
2. `st.chat_message('human')` to draw the user's input
3. `st.chat_message('assistant')` to draw the chatbot's response

This results in the following code:

```python
# Draw the chat input box
if question := st.chat_input("What's up?"):
    
    # Draw the user's question
    with st.chat_message('human'):
        st.markdown(question)

    # Generate the answer
    answer = f"""You asked: {question}"""

    # Draw the bot's answer
    with st.chat_message('assistant'):
        st.markdown(answer)
```

Check out the complete code in [app_2.py](./app_2.py).

**If you want to test and run the app:**
```bash
streamlit run step-by-step-app/app_2.py
```

## 3️⃣ Remember the chatbot interaction

In this step we'll make sure to keep track of the questions and answers so that with every redraw the history is shown.

**To do this we'll take the next steps:**
1. Add the question in a `st.session_state` called `messages`
2. Add the answer in a `st.session_state` called `messages`
3. When the app redraws, print out the history using a loop like `for message in st.session_state.messages`

This approach works because the `session_state` is stateful across Streamlit runs.

Check out the complete code in [app_3.py](./app_3.py).

**If you want to test and run the app:**
```bash
streamlit run step-by-step-app/app_3.py
```

## 4️⃣  Now for the cool part! Let's integrate with the OpenAI Chat Model 🤖

Streamlit reruns the code every time a user interacts. Because of this, we'll use data and resource caching in Streamlit so that a connection is only set up once.

We'll use `@st.cache_data()` and `@st.cache_resource()` to define caching. `cache_data` is typically used for data structures. `cache_resource` is mostly used for resources like databases.

This results in the following code to set up the Prompt and Chat Model:

```python
# Cache prompt for future runs
@st.cache_data()
def load_prompt():
    template = """You're a helpful AI assistent tasked to answer the user's questions.
You're friendly and you answer extensively with multiple sentences. You prefer to use bulletpoints to summarize.

QUESTION:
{question}

YOUR ANSWER:"""
    return ChatPromptTemplate.from_messages([("system", template)])
prompt = load_prompt()

# Cache OpenAI Chat Model for future runs
@st.cache_resource()
def load_chat_model():
    return ChatOpenAI(
        temperature=0.3,
        model='gpt-3.5-turbo',
        streaming=True,
        verbose=True
    )
chat_model = load_chat_model()
```

Instead of the static answer, we'll now switch to calling the Chain:

```python
# Generate the answer by calling OpenAI's Chat Model
inputs = RunnableMap({
    'question': lambda x: x['question']
})
chain = inputs | prompt | chat_model
response = chain.invoke({'question': question})
answer = response.content
```

Before we continue, we have to provide the `OPENAI_API_KEY` in `./streamlit/secrets.toml`. There is an example provided in `secrets.toml.example`:

```toml
# OpenAI secrets
OPENAI_API_KEY = "<YOUR-API-KEY>"
```

Fron now on, you can now start your questions-and-answer interaction with the Chatbot. Of course, as there is no integration with the Astra DB Vector Store, there will not be contextualized answers. As there is no streaming built-in yet, please give the agent a bit of time to come up with the complete answer at once.

Let's start with the question:

    What does Daniel Radcliffe get when he turns 18?

As you will see, you'll receive a very generic answer without the information that is available in the CNN data.

Check out the complete code in [app_4.py](./app_4.py).

**If you want to test and run the app:**
```bash
streamlit run step-by-step-app/app_4.py
```

## 5️⃣ Combine with the Astra DB Vector Store for additional context

Now things become really interesting! In this step we'll integrate the Astra DB Vector Store in order to provide context in real-time for the Chat Model. Steps taken to implement Retrieval Augmented Generation:
1. User asks a question
2. A semantic similarity search is run on the Astra DB Vector Store
3. The retrieved context is provided to the Prompt for the Chat Model
4. The Chat Model comes back with an answer, taking into account the retrieved context

In order to enable this, we first have to set up a connection to the Astra DB Vector Store:

```python
# Cache the Astra DB Vector Store for future runs
@st.cache_resource(show_spinner='Connecting to Astra')
def load_retriever():
    # Connect to the Vector Store
    vector_store = AstraDB(
        embedding=OpenAIEmbeddings(),
        collection_name="my_store",
        api_endpoint=st.secrets['ASTRA_API_ENDPOINT'],
        token=st.secrets['ASTRA_TOKEN']
    )

    # Get the retriever for the Chat Model
    retriever = vector_store.as_retriever(
        search_kwargs={"k": 5}
    )
    return retriever
retriever = load_retriever()
```

The only other thing we need to do is alter the Chain to include a call to the Vector Store:

```python
# Generate the answer by calling OpenAI's Chat Model
inputs = RunnableMap({
    'context': lambda x: retriever.get_relevant_documents(x['question']),
    'question': lambda x: x['question']
})
```

Before we continue, we have to provide the `ASTRA_API_ENDPOINT` and `ASTRA_TOKEN` in `./streamlit/secrets.toml`. There is an example provided in `secrets.toml.example`:

```toml
# Astra DB secrets
ASTRA_API_ENDPOINT = "<YOUR-API-ENDPOINT>"
ASTRA_TOKEN = "<YOUR-TOKEN>"
```

If we ask the same question again:

    What does Daniel Radcliffe get when he turns 18?

As you will see, now you'll receive a very contextual answer as the Vector Store provides relevant CNN data to the Chat Model.

Check out the complete code in [app_5.py](./app_5.py)

**If you want to test and run the app:**
```bash
streamlit run step-by-step-app/app_5.py
```

## 6️⃣ Finally, let's make this a streaming app

How cool would it be to see the answer appear on the screen as it is generated! Well, that's easy.

First of all, we'll create a Streaming Call Back Handler that is called on every new token generation as follows:

```python
# Streaming call back handler for responses
class StreamHandler(BaseCallbackHandler):
    def __init__(self, container, initial_text=""):
        self.container = container
        self.text = initial_text

    def on_llm_new_token(self, token: str, **kwargs):
        self.text += token
        self.container.markdown(self.text + " ")
```

Then we explain the Chat Model to make user of the StreamHandler:

```python
response = chain.invoke({'question': question}, config={'callbacks': [StreamHandler(response_placeholder)]})
```

The `response_placeholer` in the code above defines the place where the tokens need to be written. We can create that space by callint `st.empty()` as follows:

```python
# UI placeholder to start filling with agent response
with st.chat_message('assistant'):
    response_placeholder = st.empty()
```

Check out the complete code in [app_6.py](./app_6.py).

**If you want to test and run the app:**
```bash
streamlit run step-by-step-app/app_6.py
```

Now you'll see that the response will be written in real-time to the browser window.

## 7️⃣ Now let's make magic happen! 🦄

The ultimate goal of course is to add your own company's context to the agent. In order to do this, we'll add an upload box that allows you to upload PDF files which will then be used to provide a meaningfull and contextual response!

First we need an upload form which is simple to create with Streamlit:

```python
# Include the upload form for new data to be Vectorized
with st.sidebar:
    with st.form('upload'):
        uploaded_file = st.file_uploader('Upload a document for additional context', type=['pdf'])
        submitted = st.form_submit_button('Save to Astra DB')
        if submitted:
            vectorize_text(uploaded_file)
```

Now we need a function to load the PDF and ingest it into Astra DB while vectorizing the content.

```python
# Function for Vectorizing uploaded data into Astra DB
def vectorize_text(uploaded_file, vector_store):
    if uploaded_file is not None:
        
        # Write to temporary file
        temp_dir = tempfile.TemporaryDirectory()
        file = uploaded_file
        temp_filepath = os.path.join(temp_dir.name, file.name)
        with open(temp_filepath, 'wb') as f:
            f.write(file.getvalue())

        # Load the PDF
        docs = []
        loader = PyPDFLoader(temp_filepath)
        docs.extend(loader.load())

        # Create the text splitter
        text_splitter = RecursiveCharacterTextSplitter(
            chunk_size = 1500,
            chunk_overlap  = 100
        )

        # Vectorize the PDF and load it into the Astra DB Vector Store
        pages = text_splitter.split_documents(docs)
        vector_store.add_documents(pages)  
        st.info(f"{len(pages)} pages loaded.")
```

Check out the complete code in [app_final.py](app_final.py).

**If you want to test and run the app:**
```bash
streamlit run app_final.py
```

Now upload a PDF document (the more the merrier) that is relevant to you and start asking questions about it. You'll see that the answers will be relevant, meaningful and contextual! 🥳 See the magic happen!

![end-result](/./assets/first_login.png)

# Optional

## 8️⃣ Let's deploy this cool stuff to Streamlit cloud!
In this step we'll deploy your awesome app to the internet so everyone can enjoy your cool work and be amazed!

### Set up your Streamlit account
If you have not do so before, please set up your account on Streamlit. When you already have an account skip to the next step and deploy the app.

1. Head over to [Streamlit.io](https://streamlit.io) and clikc `Sign up`. Then select `Continue with Github`:

    ![Streamlit](/./assets/streamlit-0.png)

2. Log in using your Github credentials:

3. Now authorize Streamlit:

4. And set up your account:

### Deploy your app

On the main screen, when logged in, click `New app`.

1. When this is your first deployment, provide additional permissions:

2. Now define your application settings. Use YOUR repository name, and name the Main file path as `app.py`. Pick a cool App URL as you'll app will be deployed to that:

    ![Streamlit](/./assets//streamlit-5.png)

3. Click on Advanced, select Python 3.11 and copy-paste the contents from your `secrets.toml`. **Click Deploy!** Wait for a bit and your app is online for everyone to use!

    ![Streamlit](/./assets/streamlit-6.png)

⛔️ Be aware that this app is public and uses your OpenAI account which will incur cost. You'll want to shield it off by clicking `Settings->Sharing` in the main screen and define the email addresses that are allowed access. In order to enable this, link your Google account.