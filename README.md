<p align="center">
  <img src="docs/source/_static/shorkie_logo.png" alt="Shorkie" width="30%">
</p>

# Shorkie — documentation

User guide and API reference for **Shorkie**, a sequence-to-expression model for
budding yeast built on a fungal DNA language model.

### 📖 https://khchao.com/shorkie/

This repository holds only the documentation site. The code, models, and
reproduction pipelines live in **[calico/shorkie-paper](https://github.com/calico/shorkie-paper)**.

## What's covered

| Section | Contents |
|---|---|
| Getting started | Installation, a quick-start that scores a variant, and the three released models |
| Using Shorkie | Coverage prediction and variant-effect scoring (logSED) |
| Using Shorkie_LM | Masked-base prediction and sequence embeddings |
| Fine-tuning | Train on your own RNA-seq — including a demo you can actually run |
| Background | How the models were trained, the released data, reproducing the paper figures |
| Reference | API, FAQ, citation, license, contact |

## Building locally

```bash
python -m venv .venv && source .venv/bin/activate   # Python >= 3.12 required
pip install -r docs/requirements.txt
cd docs && make html          # -> docs/build/html/index.html
```

`make html` runs with `-W`, so any Sphinx warning fails the build.

Pushing to `main` rebuilds and redeploys via `.github/workflows/docs.yml`.

## Citation

> Chao, K.-H., Magzoub, M. M., Stoops, E. H., Hackett, S. R., Linder, J., &
> Kelley, D. R. (2025). *Predicting dynamic expression patterns in budding yeast
> with a fungal DNA language model.* bioRxiv.
> <https://doi.org/10.1101/2025.09.19.677475>

## License

Apache-2.0 — see [calico/shorkie-paper](https://github.com/calico/shorkie-paper/blob/main/LICENSE).
