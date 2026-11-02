# TextRelevancy

TextRelevancy tells you if a text is relevant to you and with what probability. For example:

```json
{ "relevant": true, "result": 0.988296 }
```

I developed it for [NeuroMorpho.Org](https://neuromorpho.org) during my PhD at George Mason University. There it is used to find scientific papers that describe digitally reconstructed neurons, and it has been running in production since 2020 as part of the NeuroMorpho.Org literature pipeline ([PaperBot](https://github.com/NeuroMorphoOrg/PaperBot) and its [production version](https://github.com/p-maraver/NeuroMorphoLiterature)). It was designed to be re-trained, so it can be used with any type of text.

The problem it solves: when you search for scientific literature, platforms like Google Scholar return hundreds of results, sorted by an algorithm that decides for you what is interesting, and around 90% of them are irrelevant. With TextRelevancy you give it your own texts labelled as Positive or Negative, and it trains a deep learning model that learns what is relevant for you.

The method and the results are described in this paper:
Maraver P, Tecuatl C, Ascoli GA. [Automatic identification of scientific publications describing digital reconstructions of neural morphology](https://doi.org/10.1186/s40708-023-00202-x). *Brain Informatics* 10, 23 (2023).

## Results at NeuroMorpho.Org

The model was trained with around 5,000 papers labelled by experts and reached 94.2% accuracy on a held-out test set, also checked with 10-fold cross-validation.

Not all errors cost the same, so the pipeline uses decision thresholds. Papers classified with high confidence are processed automatically: 96% of the irrelevant ones and 83% of the relevant ones, with an error of 3% or less. Only around 5 in 100 papers still need a person to review them. The service classifies around 900 papers per hour.

## How it works

1. From each text it keeps only the paragraphs that contain at least one keyword. This reduces noise and improves the results, because the model only sees the parts that could be of interest.
2. The selected paragraphs are converted into vectors with a one-hot (binary) bag of words. In the paper I compared it with count, TF-IDF and word embeddings, and one-hot worked best for this task.
3. A neural network in Keras (two hidden layers of 64 units and a sigmoid output) returns the probability that the text is relevant.

It is written in Python, uses Keras on top of TensorFlow for the model and spaCy for the natural language processing, and runs as a Docker container with a REST API.

## Install

You need [Docker](https://docs.docker.com/engine/install/). Then:

```bash
docker build -t classifier .
docker compose up
```

The API runs on port 8191.

## Configuration

The tool reads its data from files in `resources/`. This folder is shared between Docker and your computer, so you can add files before or after launching it. The data I used to train the NeuroMorpho.Org model is included as an example; you can replace it with your own.

### Labelled texts

`resources/data/{version}/` contains one JSON file per text. The file name gives the class (Positive or Negative), and inside there is the list of paragraphs of that text. `{version}` is the name of the model you want to train, so you can keep several versions.

```
Negative_1.json
Negative_2.json
Positive_1.json
Positive_2.json
```

```json
[
  "Axonal degeneration is controlled by the TIR domain NADase SARM1. ...",
  "Main text",
  "Mechanisms to destroy and clear neuronal processes are important for ..."
]
```

### Keywords for the classifier

`resources/classifier/keywords/` contains files with lists of keywords that you can change. The classifier only uses the paragraphs that contain at least one of them.

```
branching
camera lucida
cell count
colocalization
```

### Keywords for the NLP (optional)

`resources/nlp/keywords/` works the same way, but these lists are independent from the classifier (for example `cellTypes` or `tracingSystem`). The `/nlp` endpoint returns which of these words appear in a text, for example if a paper mentions *mouse*.

## API

| Method | Endpoint | What it does |
|---|---|---|
| `POST` | `/classifier/classify?version={version}` | Receives the list of paragraphs of a text and returns if it is relevant and the probability |
| `PUT` | `/classifier/train?version={version}` | Prepares the training texts: extracts the paragraphs with keywords |
| `POST` | `/classifier/train?version={version}` | Trains the model and returns the accuracy and loss |
| `GET` | `/nlp` | Returns the NLP keywords found in a text |

The full documentation is in the code comments (apiDoc).

## Author

Patricia Maraver. I designed, trained and deployed the classifier during my PhD in Bioengineering at George Mason University (Krasnow Institute), in collaboration with NeuroMorpho.Org.

## License

[AGPL-3.0](LICENSE)

## Configuration

All inputs live in `resources/`. The NeuroMorpho.Org training data is included as an example; replace it with your own.

### Labelled texts: `resources/data/<version>/`

One JSON file per text. The file name gives the class (`Positive` or `Negative`); each file holds the list of paragraphs of that text. `<version>` lets you keep several trained models side by side.

```
Negative_1.json
Negative_2.json
Positive_1.json
Positive_2.json
```

```json
[
  "Axonal degeneration is controlled by the TIR domain NADase SARM1. ...",
  "Main text",
  "Mechanisms to destroy and clear neuronal processes are important for ..."
]
```

### Classifier keywords: `resources/classifier/keywords/`

Each file holds a customisable list of keywords. A paragraph is passed to the model only if it contains at least one of them.

```
branching
camera lucida
cell count
colocalization
```

### NLP keywords (optional): `resources/nlp/keywords/`

Independent keyword lists (e.g. `cellTypes`, `tracingSystem`). The `/nlp` endpoint returns which of them appear in a text, for example whether a paper mentions *mouse*.

## API

Documented with apiDoc in the code.

| Method | Endpoint | What it does |
|---|---|---|
| `POST` | `/classifier/classify?version=<v>` | Body: list of paragraphs. Returns whether any paragraph has relevant keywords and the probability of relevance |
| `PUT` | `/classifier/train?version=<v>` | Pre-processes the training texts: extracts the relevant paragraphs |
| `POST` | `/classifier/train?version=<v>` | Vectorises the texts, trains the model and returns accuracy and loss |
| `GET` | `/nlp` | Returns the NLP keywords found in a text |

**Stack:** Python 3.8, Keras / TensorFlow 2.3, spaCy 2.3, scikit-learn, Falcon, Gunicorn, Docker.

## Author

**Patricia Maraver**: design, training and deployment, during my PhD in Bioengineering at George Mason University (Krasnow Institute), in collaboration with NeuroMorpho.Org.

## License

[AGPL-3.0](LICENSE)
