# Evaluation Setup

This file is outside the editable surface. It defines how results are judged. Agents cannot modify the evaluator or the scoring logic — the evaluation is the trust boundary.

Consider defining more than one evaluation criterion. Optimizing for a single number makes it easy to overfit and silently break other things. A secondary metric or sanity check helps keep the process honest.

eval_cores: 1
eval_memory_gb: 1.0
prereq_command: npm run prepare

## Setup

1. Install Node.js dependencies:

   ```bash
   npm install
   ```

2. Build the TypeScript source (compiles to dist/commonjs/ and dist/esm/):

   ```bash
   npm run prepare
   ```

   This runs `tshy` (TypeScript Hybrid) to compile both CommonJS and ESM outputs, then minifies the result.

3. The test suite validates correctness:
   ```bash
   npm test
   ```

The `prereq_command` above ensures the build runs before each benchmark evaluation.

## Run command

```bash
#!/bin/bash
set -e
export CDPATH=

# Create benchmark fixture
wd=$PWD
mkdir -p "$wd/bench-working-dir/eval-fixture"
cd "$wd/bench-working-dir/eval-fixture"

if ! [ -d "0" ]; then
  # Create test fixture: 1000 directories with 10 files each
  for i in {0..9}; do
    for j in {0..9}; do
      for k in {0..9}; do
        mkdir -p "$i/$j/$k"
        for f in {0..9}; do
          touch "$i/$j/$k/$f.txt"
        done
      done
    done
  done
fi

cd "$wd/bench-working-dir/eval-fixture"

# Warm filesystem cache
node -e "const {globSync} = require('$wd/dist/commonjs/index.js'); globSync('**/*.txt');" &>/dev/null || true

# Time globSync on **/*.txt pattern
start=$(date +%s%N)
count=$(node -e "const {globSync} = require('$wd/dist/commonjs/index.js'); console.log(globSync('**/*.txt').length);")
end=$(date +%s%N)

# Calculate throughput (files matched per second)
duration_ns=$((end - start))
duration_s=$(echo "scale=6; $duration_ns / 1000000000" | bc)
ops_per_sec=$(echo "scale=2; $count / $duration_s" | bc)

echo "METRIC=$ops_per_sec"
```

## Output format

The benchmark prints `METRIC=<number>` to stdout, where the number represents files matched per second.

## Metric parsing

The CLI looks for `METRIC=<number>` or `ops_per_sec=<number>` in the output.

## Ground truth

The baseline metric measures glob throughput: how many files per second the `globSync('**/*.txt')` call can match in a test fixture of 10,000 text files across 1,000 nested directories.

The metric captures:

- Pattern matching speed (minimatch integration)
- Filesystem traversal efficiency (path-scurry usage)
- Directory walking performance
- Result collection overhead

Higher values indicate better performance. A 1% improvement (metric_tolerance: 0.01) represents meaningful optimization given the system call and I/O overhead inherent in filesystem operations.
