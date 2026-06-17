# Explore the evolution traces interactively

Each design's full search tree (every candidate, its parent, diff, score, and PPA
metrics) is a ShinkaEvolve SQLite DB. Walk any of them with the shinka visualiser:

```bash
pip install shinka
shinka_visualize ../data/adder_8bit.sqlite     # opens the tree/code explorer in a browser
```

Four exemplar DBs are bundled under `data/` (adder_8bit, fsm, div_16bit, multi_8bit —
the designs discussed in the README). The **full 45-design trace set** (1,783 candidate
rows, tree-traceable via `id`/`parent_id`, joined to each RTLLM problem) is on
HuggingFace: https://huggingface.co/datasets/EvanOLeary/rtllm-shinka-evolve

For a static walk-through (no install), see the per-design pages under `designs/` —
each is the reference → best lineage with a code diff per edge.
