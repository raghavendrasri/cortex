# Predictor

A Predictor is a Python file that describes how to initialize a model and use it to make a prediction.

The lifecycle of a replica running a Predictor starts with loading the implementation file and executing code in the global scope. Once the implementation is loaded, Cortex calls the `init()` function to allow for any additional preparations. The `init()` function is typically used to download and initialize the model. It receives the metadata object, which is an arbitrary dictionary defined in the API configuration (it can be used to pass in the path to the exported/pickled model, vocabularies, aggregates, etc). Once the `init()` function is executed, the replica is available to accept requests. Upon receiving a request, the replica calls the `predict()` function with the JSON payload and the metadata object. The `predict()` function is responsible for returning a prediction from a sample.

Global variables can be shared across functions safely because each replica handles one request at a time.

## Implementation

```python
# initialization code and variables can be declared here in global scope

def init(model_path, metadata):
    """Called once before the API is made available. Setup for model serving such
    as downloading/initializing the model or downloading vocabulary can be done here.
    Optional.

    Args:
        model_path: Local path to model file or directory if specified by user in API configuration, otherwise None.
        metadata: Custom dictionary specified by the user in API configuration.
    """
    pass

def predict(sample, metadata):
    """Called once per request. Model prediction is done here, including any
    preprocessing of the request payload and postprocessing of the model output.
    Required.

    Args:
        sample: The JSON request payload (parsed as a Python object).
        metadata: Custom dictionary specified by the user in API configuration.

    Returns:
        A prediction
    """
```

## Example

```python
import boto3
from my_model import IrisNet

# variables declared in global scope can be used safely in both functions (one replica handles one request at a time)
labels = ["iris-setosa", "iris-versicolor", "iris-virginica"]
model = IrisNet()

def init(model_path, metadata):
    # Initialize the model
    model.load_state_dict(torch.load(model_path))
    model.eval()


def predict(sample, metadata):
    # Convert the request to a tensor and pass it into the model
    input_tensor = torch.FloatTensor(
        [
            [
                sample["sepal_length"],
                sample["sepal_width"],
                sample["petal_length"],
                sample["petal_width"],
            ]
        ]
    )

    # Run the prediction
    output = model(input_tensor)

    # Translate the model output to the corresponding label string
    return labels[torch.argmax(output[0])]
```