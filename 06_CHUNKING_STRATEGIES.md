# Chunking Strategies: Complete Guide

**Detailed guide to all chunking methods with trade-offs and examples**

---

## Table of Contents
1. [Why Chunking Matters](#why-chunking-matters)
2. [4 Chunking Strategies](#4-chunking-strategies)
3. [Trade-off Analysis](#trade-off-analysis)
4. [Implementation Guide](#implementation-guide)
5. [Interview Questions](#interview-questions)

---

## Why Chunking Matters

**The Problem:**
```
Document: 5000 words
Tokens: 6667 tokens
Max embedding size: 2048 tokens
Result: Can't embed! ❌
```

**The Solution: Chunking**
```
Split into chunks < 2048 tokens
Embed each chunk separately
Better search results
```

**Impact on Quality:**
```
No chunking: Can't embed large docs
Fixed chunking: Works but loses context
Semantic chunking: Best quality
Recursive chunking: Best for hierarchies
```

---

## 4 Chunking Strategies

### Strategy 1: Fixed Size Chunking

**Definition:** Split document into fixed-size chunks (e.g., 500 words)

```python
def chunk_fixed_size(text: str, chunk_size: int = 500, overlap: int = 50) -> List[str]:
    """Split text into fixed-size chunks with overlap"""
    words = text.split()
    chunks = []
    
    for i in range(0, len(words), chunk_size - overlap):
        chunk = ' '.join(words[i:i+chunk_size])
        chunks.append(chunk)
    
    return chunks

# Example
text = "The cat sat on the mat. The dog ran in the park..."
chunks = chunk_fixed_size(text, chunk_size=100, overlap=10)
print(f"Created {len(chunks)} chunks")
```

**Characteristics:**
```
Simplicity: ⭐⭐⭐⭐⭐ (Very simple)
Quality: ⭐⭐ (May break sentences)
Speed: ⭐⭐⭐⭐⭐ (Very fast)
Context: ⭐⭐ (Limited context)
```

**Pros:**
- ✓ Simple to implement
- ✓ Fast
- ✓ Predictable
- ✓ Works for most cases

**Cons:**
- ✗ May break sentences
- ✗ Loses semantic boundaries
- ✗ Overlap can be wasteful
- ✗ Not ideal for structured content

**Best For:**
- Blog posts
- News articles
- General documents
- When speed matters

**Example:**
```
Document: "Machine Learning is a subset of AI. 
           Deep Learning uses neural networks. 
           NLP processes text."

Chunks (100 words):
1. "Machine Learning is a subset of AI. Deep Learning uses neural networks."
2. "Deep Learning uses neural networks. NLP processes text."
```

---

### Strategy 2: Semantic Chunking

**Definition:** Split document by semantic boundaries (sections, paragraphs)

```python
def chunk_semantic(text: str) -> List[str]:
    """Split text by semantic boundaries (paragraphs, sections)"""
    # Split by double newlines (paragraphs)
    paragraphs = text.split('\n\n')
    
    chunks = []
    current_chunk = ""
    
    for para in paragraphs:
        if len(current_chunk) + len(para) < 2048:  # Max tokens
            current_chunk += para + "\n\n"
        else:
            if current_chunk:
                chunks.append(current_chunk)
            current_chunk = para + "\n\n"
    
    if current_chunk:
        chunks.append(current_chunk)
    
    return chunks

# Example
text = """
## Introduction
Machine Learning is a subset of AI.

## Deep Learning
Deep Learning uses neural networks.

## NLP
NLP processes text.
"""

chunks = chunk_semantic(text)
print(f"Created {len(chunks)} chunks")
```

**Characteristics:**
```
Simplicity: ⭐⭐⭐ (Moderate)
Quality: ⭐⭐⭐⭐ (Good)
Speed: ⭐⭐⭐⭐ (Fast)
Context: ⭐⭐⭐⭐ (Good context)
```

**Pros:**
- ✓ Respects semantic boundaries
- ✓ Better context preservation
- ✓ Cleaner chunks
- ✓ Works well for structured content

**Cons:**
- ✗ Requires document structure
- ✗ May create uneven chunks
- ✗ Doesn't work for unstructured text
- ✗ Requires parsing

**Best For:**
- Technical documentation
- Research papers
- Books with chapters
- Structured content

**Example:**
```
Document with sections:
## Section 1
Content here...

## Section 2
Content here...

Result: Each section becomes a chunk
```

---

### Strategy 3: Recursive Chunking

**Definition:** Split hierarchically (sentences → paragraphs → sections)

```python
def chunk_recursive(text: str, chunk_size: int = 500, overlap: int = 50) -> List[str]:
    """Recursively split text maintaining semantic boundaries"""
    
    def split_text(text, separators):
        if not separators:
            return [text]
        
        separator = separators[0]
        splits = text.split(separator)
        
        # Recursively split with next separator
        good_splits = []
        for split in splits:
            if len(split) < chunk_size:
                good_splits.append(split)
            else:
                # Recursively split with next separator
                sub_splits = split_text(split, separators[1:])
                good_splits.extend(sub_splits)
        
        return good_splits
    
    # Try separators in order
    separators = ["\n\n", "\n", ". ", " "]
    splits = split_text(text, separators)
    
    # Merge splits into chunks
    chunks = []
    current_chunk = ""
    
    for split in splits:
        if len(current_chunk) + len(split) < chunk_size:
            current_chunk += split
        else:
            if current_chunk:
                chunks.append(current_chunk)
            current_chunk = split
    
    if current_chunk:
        chunks.append(current_chunk)
    
    return chunks

# Example
text = """
## Chapter 1
Introduction paragraph.

More content here.

## Chapter 2
Another section.
"""

chunks = chunk_recursive(text)
print(f"Created {len(chunks)} chunks")
```

**Characteristics:**
```
Simplicity: ⭐⭐ (Complex)
Quality: ⭐⭐⭐⭐⭐ (Excellent)
Speed: ⭐⭐⭐ (Moderate)
Context: ⭐⭐⭐⭐⭐ (Excellent)
```

**Pros:**
- ✓ Maintains semantic boundaries
- ✓ Excellent context preservation
- ✓ Works for any document type
- ✓ Produces clean chunks
- ✓ Handles nested structures

**Cons:**
- ✗ More complex to implement
- ✗ Slower than fixed size
- ✗ Requires tuning separators
- ✗ May create uneven chunks

**Best For:**
- Mixed content (structured + unstructured)
- Complex documents
- When quality matters most
- Production systems

**Example:**
```
Document:
## Section 1
Paragraph 1. Sentence 1. Sentence 2.
Paragraph 2. Sentence 3. Sentence 4.

## Section 2
Paragraph 3. Sentence 5. Sentence 6.

Process:
1. Try splitting by "\n\n" (paragraphs)
2. If too large, split by "\n" (lines)
3. If too large, split by ". " (sentences)
4. If too large, split by " " (words)

Result: Clean, semantic chunks
```

---

### Strategy 4: Sliding Window Chunking

**Definition:** Overlapping chunks with sliding window

```python
def chunk_sliding_window(text: str, window_size: int = 500, step: int = 250) -> List[str]:
    """Create overlapping chunks using sliding window"""
    words = text.split()
    chunks = []
    
    for i in range(0, len(words), step):
        chunk = ' '.join(words[i:i+window_size])
        if chunk:
            chunks.append(chunk)
    
    return chunks

# Example
text = "word1 word2 word3 ... word1000"
chunks = chunk_sliding_window(text, window_size=100, step=50)
print(f"Created {len(chunks)} chunks with overlap")
```

**Characteristics:**
```
Simplicity: ⭐⭐⭐ (Moderate)
Quality: ⭐⭐⭐⭐ (Good)
Speed: ⭐⭐⭐⭐ (Fast)
Context: ⭐⭐⭐⭐⭐ (Excellent)
```

**Pros:**
- ✓ Excellent context preservation
- ✓ No information loss at boundaries
- ✓ Better search results
- ✓ Handles edge cases well

**Cons:**
- ✗ Creates duplicate content
- ✗ Higher storage cost
- ✗ More embeddings to generate
- ✗ Slower than fixed size

**Best For:**
- When context is critical
- Search quality is priority
- When cost allows
- Dense documents

**Example:**
```
Text: "A B C D E F G H I J"
Window size: 3, Step: 1

Chunks:
1. "A B C"
2. "B C D"
3. "C D E"
4. "D E F"
...

Overlap: 2 words between chunks
```

---

## Trade-off Analysis

### Quality vs Speed

```
Fixed Size:
- Speed: ⭐⭐⭐⭐⭐ (Fastest)
- Quality: ⭐⭐ (Lowest)
- Use when: Speed matters

Semantic:
- Speed: ⭐⭐⭐⭐ (Fast)
- Quality: ⭐⭐⭐⭐ (Good)
- Use when: Structured content

Recursive:
- Speed: ⭐⭐⭐ (Moderate)
- Quality: ⭐⭐⭐⭐⭐ (Best)
- Use when: Quality matters

Sliding Window:
- Speed: ⭐⭐⭐⭐ (Fast)
- Quality: ⭐⭐⭐⭐ (Good)
- Use when: Context matters
```

### Cost vs Quality

```
Fixed Size (100 words):
- Chunks: Many
- Cost: High (many embeddings)
- Quality: Low

Fixed Size (500 words):
- Chunks: Medium
- Cost: Medium
- Quality: Medium

Recursive (500 words):
- Chunks: Medium
- Cost: Medium
- Quality: High

Sliding Window (500 words, 50% overlap):
- Chunks: Many
- Cost: High (overlap)
- Quality: Very High
```

### Chunk Size Impact

```
Small chunks (100 words):
- Pros: Precise search, low latency
- Cons: More embeddings, higher cost, less context

Medium chunks (500 words):
- Pros: Balanced cost/quality
- Cons: May lose some context

Large chunks (1000 words):
- Pros: Lower cost, more context
- Cons: Less precise search, slower
```

---

## Implementation Guide

### Step 1: Choose Strategy

```
Document Type?
├─ Unstructured (blog, news)
│  └─ Use: Fixed Size or Sliding Window
│
├─ Structured (docs, papers)
│  └─ Use: Semantic or Recursive
│
└─ Mixed
   └─ Use: Recursive
```

### Step 2: Set Parameters

```
Chunk Size:
- Small (<250 words): Precise but expensive
- Medium (250-500): Balanced
- Large (>500): Cheap but less precise

Overlap:
- None: Fastest, may lose context
- 10-20%: Balanced
- 50%: Best context, expensive
```

### Step 3: Test & Measure

```python
def evaluate_chunking(chunks, queries):
    """Evaluate chunking quality"""
    
    # Measure chunk size distribution
    sizes = [len(c.split()) for c in chunks]
    print(f"Avg chunk size: {sum(sizes)/len(sizes):.0f} words")
    print(f"Min: {min(sizes)}, Max: {max(sizes)}")
    
    # Measure search quality
    accuracy = 0
    for query in queries:
        # Find relevant chunk
        # Compare with ground truth
        pass
    
    print(f"Search accuracy: {accuracy:.1%}")
```

---

## Interview Questions

**Q1: Why do we need chunking?**
```
Answer:
- Max embedding size: 2048 tokens
- Documents often > 2048 tokens
- Can't embed full documents
- Solution: Split into chunks < 2048 tokens
- Benefit: Better search results
```

**Q2: What's the best chunking strategy?**
```
Answer:
- Depends on document type
- Unstructured: Fixed size or sliding window
- Structured: Semantic or recursive
- General recommendation: Recursive
  - Works for any document
  - Maintains semantic boundaries
  - Produces clean chunks
```

**Q3: How do you choose chunk size?**
```
Answer:
- Small (100 words): Precise but expensive
- Medium (500 words): Balanced (RECOMMENDED)
- Large (1000 words): Cheap but less precise

Factors:
- Document type
- Search precision needed
- Cost budget
- Latency requirements
```

**Q4: What about overlap?**
```
Answer:
- No overlap: Fastest, may lose context
- 10-20% overlap: Balanced
- 50% overlap: Best context, 2x cost

Trade-off:
- More overlap = Better search
- More overlap = Higher cost
- Recommended: 10-20% for balance
```

---

## Key Takeaways

1. **Fixed Size** - Simple, fast, lower quality
2. **Semantic** - Good for structured content
3. **Recursive** - Best quality, works for any document
4. **Sliding Window** - Best context preservation

**Recommendation:** Use Recursive chunking with 500-word chunks and 10-20% overlap

