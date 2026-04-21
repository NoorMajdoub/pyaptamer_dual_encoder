# Apta-MLM: Protein-Conditioned Aptamer Generation

## The Problem

The tas is given a target protein, generate a DNA/RNA aptamer sequence that binds to it.

Two core challenges make this hard:

1. **Data scarcity** — known protein-aptamer binding pairs are rare(exp of the dataset that has ~725 positive (binding) pairs vs ~2175 negative pairs)

2. **No aptamer foundation model** — ESM2 is pretrained on proteins, 
NucleotideTransformer on DNA/RNA. Neither was trained on protein-aptamer 
interactions. Forcing both modalities through one model (like PepMLM does) 
creates a vocabulary and semantic mismatch.
## summary on RNA BaNg
BAnG solves the same problem of generating RNA/DNA sequences that bind a target protein — but takes a very different approach to generation.
"Their core insight: aptamers don't bind randomly along their length. They have a short binding motif (usually ~6 nucleotides) that does the actual work, embedded somewhere inside a longer flanking context. So instead of generating left-to-right like a normal language model, they start generation from the motif outward in both directions simultaneously.
"
Their approach:

Pick an anchor point (a known or predicted binding residue) and insert two special tokens <ancl><ancr> there
Generate one token to the right, then one to the left, alternating outward (Bidirectional)
Stop when <eos> is produced in both directions
The protein conditions the generation via cross-attention from a protein module into the nucleotide module
I did not fully implement this , in this first version I just rebuilt a simplified version more oriented to the issue #131

Protein module: self-attention + geometric attention (uses AlphaFold2 3D backbone frames, not just sequence)
Nucleotide module: self-attention + cross-attention to protein representation + feedforward, with a custom bidirectional BAnG attention mask replacing the usual causal mask
<img width="808" height="614" alt="image" src="https://github.com/user-attachments/assets/0d01e5fb-c3ea-444d-ac4e-3d14863d9d92" />

## summary on PepMLM

PepMLM solves a related problem — generating peptide binders for a target protein.

Their approach:
1. Format input as `[START] p1...pn [MASK]*n [END]`  
   where `p1...pn` is the target protein, `[MASK]*n` is the placeholder 
   for the peptide to generate
2. Encode with ESM2 (pretrained protein encoder)
3. Decode greedily or with top-k sampling

Works well for peptides because ESM2 speaks amino acids fluently. 
For DNA/RNA aptamers, this breaks — ESM2 has no understanding of nucleotide sequences.

## Proposed Approach: Dual Encoder with Cross-Attention

Instead of forcing both modalities through one model, use two domain-specific encoders:

- **Protein encoder**: frozen ESM2 — understands proteins
- **Nucleotide encoder**: frozen NucleotideTransformer — understands DNA/RNA
- **Cross-attention**: the mechanism that makes them talk to each other

The architecture is inspired by BAnG (Bidirectional Anchored Generation for 
Conditional RNA Design, 2025) which proposes a dual-encoder model with 
cross-attention between protein and nucleotide representations.

Adaptations made from BAnG:
- Replaced geometric attention (requires 3D protein structure) with regular self-attention 
- Added MLM training objective on the aptamer side, conditioned on the protein
- Both encoders kept frozen due to data constraints

Training objective: mask x% of aptamer tokens, predict them given the full protein (tested with 15% for now)

## Handling Data issue

725 positive pairs is small. 2175 negative pairs.

Proposed addition: contrastive loss(infoloss) alongside MLM loss.
- Positive pairs (protein, binding aptamer) → pulled together in embedding space
- Negative pairs (protein, non-binding aptamer) → pushed apart
Or BaNG their two-stage training idea.


## Still not sure about

**On evaluation:**  
PepMLM validated that low pseudo-perplexity (PPL) correlates with good AlphaFold-Multimer scores (ipTM, pLDDT). This is their "sanity check" — 
low PPL isn't just a statistical artifact, it genuinely predicts binding.

For aptamers, this translation is not straightforward:
- AlphaFold-Multimer metrics apply to protein-peptide complexes, not nucleic acids
- What is the equivalent structural validation tool for protein-aptamer complexes?
  
**On input format:**  
The current pyaptamer loaders expect PDB files (3D structure).  
BAnG also uses 3D protein structure via AlphaFold2.  
PepMLM works with 1D sequence .

**On sequence length:**  
BAnG notes that rna/dna length is often unknown in advance, which is why they use autoregressive generation. This implementation currently requires a fixed length(like in pepmlm)

## Current State

- Architecture implemented and forward pass tested
- MLM training step implemented on dummy data
- Contrastive loss: in progress (requires pooling layer for sequence-level embeddings and I am bit confused)

## References

- BAnG: Bidirectional Anchored Generation for Conditional RNA Design (2025)
- PepMLM: Target Sequence-Conditioned Generation of Peptide Binders via Masked Language Modeling
