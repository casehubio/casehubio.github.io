---
layout: post
title: "Language families and the MatchingStrategy SPI"
date: 2026-11-27
entry_type: note
subtype: diary
projects: [casehubio/platform]
tags: [nlp, i18n, template-registry, spi-design]
series: issue-965-multi-lang-nlp-tokenizers
---

The unified template registry (#959) shipped with an English-only NLP pipeline — `OpenNlpTokenizer` feeding `StructuralMatcher` feeding `DiscriminationTrie`. It works. The question was whether adding French, Spanish, and Italian was just a matter of swapping in different OpenNLP model files. It isn't.

## The four assumptions

`StructuralMatcher` does a sequential token walk: literal at position 0, variable at position 1, literal at position 2. That walk makes four assumptions that hold for English but collapse for other language families.

**Tokens are space-delimited.** Chinese and Japanese have no word boundaries. Segmentation — deciding where one word ends and the next begins — is a fundamentally different problem from tokenization.

**Lemmatisation reduces to a base form.** Turkish is agglutinative. A single token like *yapamayacaklarımızdan* encodes subject, object, tense, mood, negation, and evidentiality. OpenNLP's lemmatiser produces one lemma per token — it can't decompose that into constituent morphemes. You need Zemberek for Turkish, HFST for Finnish.

**Word order matches the template.** Japanese is SOV. Arabic is VSO. German subordinate clauses push the verb to the end. A sequential walk that expects "I approve this design" can't match "このデザインを承認します" even with perfect tokenization — the structural elements appear in a different order.

**One token = one semantic unit.** German compound nouns fuse multiple concepts into a single token. *Handelsregistereintragung* is one token that means "commercial register entry" — three words packed into one.

## Strategy, not configuration

I considered three approaches: parameterising `StructuralMatcher` with language modes, wrapping it with pre/post-processing adapters, or replacing it with a strategy pattern. The first two fail on CJK. Character n-gram indexing for Chinese and morpheme-set matching for Turkish share nothing with the sequential walk — they're different algorithms behind the same interface.

So: `MatchingStrategy` as an SPI. Each language family provides its own implementation with its own candidate selection mechanism and its own matching algorithm. The existing `StructuralMatcher` becomes the SVO family's internal implementation. The positional `DiscriminationTrie` becomes the SVO family's candidate selector. Other families bring their own.

The SPI uses an opaque `Index` handle — the strategy builds whatever internal data structure it needs and returns a handle the registry stores and passes back for matching. This keeps SVO-specific types (trie nodes, positional elements) out of the API module entirely. The registry never touches NLP-specific intermediate representations.

```java
public interface MatchingStrategy {
    String familyId();
    Index buildIndex(TemplateNamespace namespace, NlpTokenizer tokenizer);
    List<PatternRef> selectCandidates(Index index, List<AnnotatedToken> inputTokens);
    Optional<MatchResult> match(Index index, PatternRef candidate,
                                 List<AnnotatedToken> inputTokens);
    interface Index {}
}
```

## The OpenNLP gap

The initial design assumed OpenNLP could handle everything — just swap model files per language. Claude's design review caught the flaw.

OpenNLP provides tokenization, POS tagging, lemmatization, NER, sentence detection, and chunking. It does not provide morpheme decomposition. Its UD tokenizer models for CJK are trained on pre-segmented treebank data — they assume word boundaries already exist. For real-world unsegmented Chinese text, Jieba or HanLP is demonstrably more accurate. For Japanese, Kuromoji (the same analyzer Lucene and Elasticsearch use) is the standard. For Turkish morphological analysis, Zemberek is the only serious option.

This meant the original scope — full implementation of all language families — was speculative engineering. I was designing implementations around tools that couldn't deliver what the design required.

## Scope narrowing

The revised scope: design the SPI informed by all language families, but implement only the SVO family. English plus Romance languages (French, Spanish, Italian, Portuguese) share enough linguistic structure — space-delimited tokens, SVO word order, inflectional rather than agglutinative morphology — that the existing `StructuralMatcher` algorithm works for all of them with different model files.

CJK, agglutinative, VSO, and Germanic (German's compound splitting is a question mark) each become separate issues with their own library research and their own `MatchingStrategy` implementation.

This brought the scale back from L/High to M/Med. The SPI design is the hard part — it needs to accommodate algorithms that don't exist yet. The SVO implementation is mechanical: move four existing classes to a new module, wrap them in `SvoMatchingStrategy`, add per-language stop words and an `OpenNlpTokenizerFactory` that caches loaded models by language tag.

## What the SPI accommodates

The design review forced a useful exercise: stress-testing each SPI method against the deferred families. `buildIndex` takes a `TemplateNamespace` and an `NlpTokenizer` — a CJK strategy could ignore the tokenizer entirely and use Kuromoji internally. `selectCandidates` returns `PatternRef` objects — a bag-of-morphemes selector for Turkish would return the same type. `match` returns `Optional<MatchResult>` with a score — the registry applies the threshold, not the strategy, so different families can normalise scores differently.

The diagnostic methods (`matchWithTrace`, `traceAll`, `structuralSummaries`) are default methods that return empty results. Strategies opt in when they support diagnostics. SVO implements all three; a future CJK strategy might not support element-level tracing at all.

One concrete consequence: `LemmatisedKeywordFilter` moves to the SVO module with per-language stop word sets. The hardcoded English stop words (`"a", "an", "the", "is", "are"...`) obviously fail for French ("le", "la", "les", "de", "du", "des"). Each SVO language gets its own stop word set sourced from Lucene's analyzers.

## The plan

Three batches. SPI types in `template-registry-api`. New `template-registry-nlp-svo` module with the moved classes and strategy implementation. Registry refactoring to delegate through the strategy layer. Eight tasks, all TDD. Ready for a fresh session to execute.

The interesting part isn't this implementation — it's what ships next. The SPI is designed so that adding Japanese template matching means writing a `CjkMatchingStrategy` and dropping it on the classpath. No registry changes. No API changes. The same opaque `Index` pattern, the same `PatternRef` and `MatchResult` types. Just a different algorithm behind the interface.
