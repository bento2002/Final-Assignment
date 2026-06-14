# Injection Detection Model

## Plan
### Injection Types we will deal with:
 - zero size fonts
 - white text + layer masking (pdf)
 - instruction in image metadata
 - text overlay in images (OCR)

### Pipeline creation
Common user's steps:
1) Data type choice: PDF, JPEG, PNG
2) Data Upload
3) Backend: Model choice corresponding with data type choice.
4) Backend: File scan and comparion to potential prompt injection techniques.
5) Backend - If injected: What's the purpose? what is the injection doing?
6) Backend - Run sturctured prompt using another LLM model to find out what the prompt is engineered to do?


### Data Generation
1) Different type of prompt injections

### Questions for Sahar
 - Do we need to generate both the PDF and the JPEG and PNG or just the prompt?
 - How many rows of data?
 - Do we need all data types or to focus on one type?
 - Do we need to reccomend or can we just run cosine similarities for the prompt match?
 - In the generation output do we need to actually use our model to generate something new or can we use an external gpt model with a prompt we're structuring?
 - Do we need to use a HF model to generate our synthetic data or can we use models from grok?