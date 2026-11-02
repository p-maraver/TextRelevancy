# TextRelevancy

**Deep-learning classifier that tells you whether a text is relevant to you, and with what probability.** Trained on your own labelled examples, so the model is fully personalised to your domain.

```json
{ "relevant": true, "result": 0.988296 }
```

It was built for [NeuroMorpho.Org](https://neuromorpho.org), where it decides whether a scientific paper describes digital reconstructions of neurons. It is the ML component of the NeuroMorpho.Org literature pipeline ([PaperBot](https://github.com/NeuroMorphoOrg/PaperBot) and its [production deployment](https://github.com/p-maraver/NeuroMorphoLiterature)), in production since 2020. It can be re-trained for any type of text.

**Why:** search engines such as Google Scholar return hundreds of results ranked by a black-box algorithm, and around 90% of them are irrelevant to a specialised curation task. TextRelevancy learns what *you* consider relevant from your own positive and negative examples.

Paper: Maraver P, Tecuatl C, Ascoli GA. [Automatic identification of scientific publications describing digital reconstructions of neural morphology](https://doi.org/10.1186/s40708-023-00202-x). *Brain Informatics* 10, 23 (2023).

## Results at NeuroMorpho.Org

- **94.2% accuracy** on a held-out test set, confirmed with 10-fold cross-validation, trained on ~5,000 expert-labelled papers.
- Decision thresholds calibrated to the cost of each error: high-confidence predictions automate **96% of irrelevant and 83% of relevant** papers at ≤3% error, so only ~5 in 100 papers need human review.
- ~900 papers classified per hour.

## How it works

1. **Paragraph selection:** keep only the paragraphs that contain domain keywords (spaCy). This cuts noise and input size, and improves performance.
2. **Vectorisation:** one-hot (binary) bag-of-words. In the paper, it beat count, TF-IDF and word-embedding representations for this task.
3. **Model:** feed-forward network in Keras / TensorFlow, with two hidden layers of 64 units and a sigmoid output giving the probability of relevance.
4. **Decision:** the calling pipeline applies the calibrated thresholds; only uncertain texts go to a human.

## Run it

Requires [Docker](https://docs.docker.com/engine/install/).

```bash
docker build -t classifier .
docker compose up
```

The service exposes a REST API on port 8191. The `resources/` folder is shared between Docker and the host, so you can add or replace data before or after launching.

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
