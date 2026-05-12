# SNOMED CT // Lookup Benchmark

A single-file visual demo contrasting three SNOMED CT concept lookup approaches:

- **Terminology server (FHIR `$lookup`)** — network round-trip (~60 ms)
- **In-memory B-tree** — RAM-resident traversal (~70 μs)
- **Compiled MPHF** — direct array lookup (~0.4 μs)

Toggle the left panel between the terminology server and the B-tree mode; the right panel always shows the compiled minimal perfect hash function.

## Deployed on 

https://debojadebayo.github.io/snomed-demo/


## Author

Built by [@debojadebayo](https://linkedin.com/in/debojadebayo).
