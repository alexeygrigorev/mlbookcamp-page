---

title: KFServing Transformers
layout: article
image: img/articles/cover/kfserving-tranformers.jpg

---

We’ll cover:

- Setting up a transformer from scratch
- Using existing transformers
- Code: [https://github.com/alexeygrigorev/kubeflow-deep-learning](https://github.com/alexeygrigorev/kubeflow-deep-learning)


Prerequisites:

- Configured KFServing cluster (see [Creating a KFServing Cluster on EKS](/article/kfserving-eks-install) for more details)


## KFServing transorfmers

In KFServing, transformers sit between the client and the model
and do the transformation. The client will only need to supply the URLs
for the images.

<img src="/img/articles/kubeflow/transformer.png" />


In the transformer:

- Pre-processing: convert the URL to a NumPy array and then a list of floats
- Post-processing: convert the raw predictions to predictions with labels

In KFServing, transformers are deployed separately from the model, so
they can scale up independently. It’s good, because they do a different
kind of work — the transformer is doing I/O work (fetching the image),
while the model is doing compute work (the number crunching).

To do it, we’ll need to install the kfserving package for python:

```bash
pip install kfserving
```

To define a transformer, we need to extend the kfserving.Model class.
Let's create a python file with that ("image_transformer.py"):


```python
import argparse
import kfserving

from keras_image_helper import create_preprocessor


class ImageTransformer(kfserving.KFModel):
    def __init__(self, name, predictor_host):
        super().__init__(name)
        self.predictor_host = predictor_host
        self.preprocessor = create_preprocessor('xception', target_size=(299, 299))
        self.labels = [
            'dress',
            'hat',
            'longsleeve',
            'outwear',
            'pants',
            'shirt',
            'shoes',
            'shorts',
            'skirt',
            't-shirt'
        ]

    def image_transform(self, instance):
        url = instance['url']
        X = self.preprocessor.from_url(url)
        return X[0].tolist()

    def preprocess(self, inputs):
        instances = [self.image_transform(instance) for instance in inputs['instances']]
        return {'instances': instances}

    def postprocess(self, outputs):
        results = []

        raw = outputs['predictions']

        for row in raw:
            result = {c: p for c, p in zip(self.labels, row)}
            results.append(result)

        return {'predictions': results}


if __name__ == "__main__":
    parser = argparse.ArgumentParser(parents=[kfserving.kfserver.parser])
    parser.add_argument('--model_name',
                        help='The name that the model is served under.')
    parser.add_argument('--predictor_host',
                        help='The URL for the model predict function',
                        required=True)

    args, _ = parser.parse_known_args()

    transformer = ImageTransformer(args.model_name, predictor_host=args.predictor_host)
    kfserver = kfserving.KFServer()
    kfserver.start(models=[transformer])
```


Let's test it:

```bash
HOST="clothing-model.default.kubeflow.mlbookcamp.com"

python image_transformer.py \
    --predictor_host="${HOST}" \
    --model_name="clothing-model"
```


It runs a web service locally, and we can use it for testing the
transformer. So let’s create another file for testing it
("test-transformer.py"):

```python
import requests

data = {
    "instances": [
        {"url": "http://bit.ly/mlbookcamp-pants"},
    ]
}

url = 'http://localhost:8080/v1/models/clothing-model:predict'

result = requests.post(url, json=data).json()

print(result)
```

Run it:
```bash
python test-transformer.py
```

The output:

```python
{'predictions': [{'dress': -1.86828923, 'hat': -4.76124525,
'longsleeve': -2.31698346, 'outwear': -1.06257045, 'pants': 9.88715553,
'shirt': -2.81243205, 'shoes': -3.66628242, 'shorts': 3.20036, 'skirt':
-2.60233665, 't-shirt': -4.83504581}]}
```

Now let’s prepare a docker file for the tranformer ("transformer.dockerfile"):


```docker
FROM python:3.7-slim

RUN pip install --upgrade pip

RUN pip install kfserving>=0.2.1 \
    argparse>=1.4.0 \
    pillow==7.1.0 \
    keras_image_helper==0.0.1

COPY image_transformer.py image_transformer.py 

ENTRYPOINT ["python", "image_transformer.py"]
```

Build it:

```bash
IMAGE_LOCAL="clothing-model-transformer"
docker build -t ${IMAGE_LOCAL} -f transformer.dockerfile .
```

Authenticate with AWS cli, tag the image and push it to ECR (assuming
you created a registry “model-serving” in eu-west-1 — adjust it to your
case)


```bash
$(aws ecr get-login --no-include-email)

ACCOUNT=XXXXXXXXXXXX
REGISTRY=${ACCOUNT}.dkr.ecr.eu-west-1.amazonaws.com/model-serving
IMAGE_REMOTE=${REGISTRY}:${IMAGE_LOCAL}
docker tag ${IMAGE_LOCAL} ${IMAGE_REMOTE}

docker push ${IMAGE_REMOTE}
```


Before using this new transformer, let’s delete the old inference
service first:

```bash
kc delete -f tf-clothing.yaml
```

Now let’s adjust the definition:

```yaml
apiVersion: "serving.kubeflow.org/v1alpha2"
kind: "InferenceService"
metadata:
  name: "clothing-model"
spec:
  default:
    predictor:
      serviceAccountName: sa
      tensorflow:
        storageUri: "s3://mlbookcamp-models/clothing-model"
    transformer:
      custom:
        container:
          image: XXXXXXXXXXXX.dkr.ecr.eu-west-1.amazonaws.com/model-serving:clothing-model-transformer
          name: user-container
```

Apply it


```bash
kubectl apply -f tf-clothing.yaml
```

Update the url in the "test-transformer.py" script:

```python
url =
'https://clothing-model.default.kubeflow.mlbookcamp.com/v1/models/clothing-model:predict'
```

Test it:

```bash
python test-transformer.py
```

Response:

```python
{'predictions': [{'dress': -1.86828923, 'hat': -4.76124525,
'longsleeve': -2.31698346, 'outwear': -1.06257045, 'pants': 9.88715553,
'shirt': -2.81243205, 'shoes': -3.66628242, 'shorts': 3.20036, 'skirt':
-2.60233665, 't-shirt': -4.83504581}]}
```

## Using existing transformers

Instead of creating your own tranformer, you can use an existing one:

```yaml
apiVersion: "serving.kubeflow.org/v1alpha2"
kind: "InferenceService"
metadata:
  name: "clothing-model"
spec:
  default:
    predictor:
      serviceAccountName: sa
      tensorflow:
        storageUri: "s3://mlbookcamp-models/clothing-model"
    transformer:
      custom:
        container:
          image: "agrigorev/kfserving-keras-transformer:0.0.1"
          name: user-container
          env:
            - name: MODEL_INPUT_SIZE
              value: "299,299"
            - name: KERAS_MODEL_NAME
              value: "xception"
            - name: MODEL_LABELS
              value: "dress,hat,longsleeve,outwear,pants,shirt,shoes,shorts,skirt,t-shirt"
```

It uses [kfserving-keras-transformer](https://github.com/alexeygrigorev/kfserving-keras-transformer).
