# llama-prod

Deploy LLaMa models to Google Cloud Run in 5 minutes.

## Initial Setup
To play through this tutorial, I recommend creating a new project. You can do this in the console or [with the Cloud SDK](https://cloud.google.com/sdk/docs/install) (recommended). You can find your billing account id [here](https://console.cloud.google.com/billing)

```bash
cd ..
curl -O https://dl.google.com/dl/cloudsdk/channels/rapid/downloads/google-cloud-cli-425.0.0-linux-x86_64.tar.gz
tar -xf google-cloud-cli-*.tar.gz
./google-cloud-sdk/install.sh

gcloud auth login
```

Create  your new project.
```bash
export PROJECT_ID=<YOUR_UNIQUE_LOWER_CASE_PROJECT_ID>
export BILLING_ACCOUNT_ID=<YOUR_BILLING_ACCOUNT_ID>
export REGION="us-central1"
export TAG="gcr.io/$PROJECT_ID/llama-prod"
wget https://huggingface.co/eachadea/ggml-vicuna-7b-4bit/resolve/main/ggml-vicuna-7b-4bit-rev1.bin
mv ggml-vicuna-7b-4bit-rev1.bin model.bin
export MODEL=model.bin

gcloud projects create $PROJECT_ID --name="llama-prod"

# Set Default Project (all later commands will use it) 
gcloud config set project $PROJECT_ID

# Add Billing Account
gcloud beta billing projects link $PROJECT_ID --billing-account=$BILLING_ACCOUNT_ID
```

Enable the services you need.
```bash
gcloud services enable cloudbuild.googleapis.com \
    containerregistry.googleapis.com \
    run.googleapis.com
```

To try your app locally with docker simply run inside `src`:
```bash
docker build -t $TAG
docker run -p 8000:8000 $TAG
```

Again you can check it in your browser our curl it:
```bash
curl -X POST "http://localhost:8000/v1/completions" -H  "accept: application/json" -H  "Content-Type: application/json" -d "{\"prompt\":\"\n\n### Instructions:\nWhat is the capital of France?\n\n### Response:\n\",\"stop\":[\"\n\",\"###\"]}"
```

## Deployment
If everything worked out so far, we're ready to deploy our app. First we create a Google Cloud Build. Maybe it's similar to pushing a docker image to a docker registry.

```bash
gcloud builds submit --tag $TAG
```

After this is done, well it's finally time to deploy your cloud run app :).
```bash
gcloud run deploy $APP --image $TAG --platform managed --region $REGION --allow-unauthenticated
```

## Test it
Note, this may take some minutes. The URL of your app will show up in the terminal. But you can also check your app via:
```bash
# See all info about the app
gcloud run services describe $APP --region $REGION
```

Note, even though we've chosen a random port like `1234`, the deployed service will use `8080` by default. This is why we need `--port ${PORT}` in the last line of our Dockerfile.

Let's send a request to our app.
```bash
# get url
URL=$(gcloud run services describe $APP --region $REGION --format 'value(status.url)')

curl -X POST "$URL/v1/completions" -H  "accept: application/json" -H  "Content-Type: application/json" -d "{\"prompt\":\"\n\n### Instructions:\nWhat is the capital of France?\n\n### Response:\n\",\"stop\":[\"\n\",\"###\"]}"
```
Perfect! Again, we receive `{"id":"cmpl-ecd7cd52-d265-4075-a121-af852a057e43","object":"text_completion","created":1681031851,"model":"ggml-vicuna-7b-4bit-rev1.bin","choices":[{"text":"The capital of France is Paris.","index":0,"logprobs":null,"finish_reason":"stop"}],"usage":{"prompt_tokens":25,"completion_tokens":8,"total_tokens":33}}`.

## Clean up
If you don't need the app anymore remove it. (You may also delete the whole project instead.)
```bash
gcloud run services delete $APP --region $REGION

# Check if it disAPPeared (optional)
gcloud run services list
```