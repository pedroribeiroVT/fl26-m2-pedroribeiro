# M2 Technical & Design Understanding

This file is an assessed technical-understanding artifact, not ordinary project documentation.
Answer all four questions using your own submitted implementation. Concise answers are acceptable when they are technically correct and specific.

Generic descriptions of C++ concepts or restatements of the assignment that do not identify and explain corresponding parts of your code will receive limited credit.

## 1. Polymorphism and dynamic dispatch - 1.5 points

Identify one place in your M2 implementation where runtime polymorphism occurs. Name the relevant base interface, derived implementation, and `ProcessingCore` function involved. Trace the call from `ProcessingCore` to the selected strategy implementation and explain why the derived implementation is invoked.

Then explain what would change if the relevant operation were not declared `virtual`.

The place I'll trace is search. `ProcessingCore::search` runs `impl_->retrieval_strategy->search(query, k, impl_->chunks, impl_->corpus_index)`. `impl_->retrieval_strategy` is a `std::unique_ptr<RetrievalStrategy>`, so the compiler only knows the base type there. `RetrievalStrategy::search` is virtual, so the call goes through the whatever object the pointer holds. With the default constructor that object is the `RetrievalEngine` made by `std::make_unique<RetrievalEngine>()`. With the 3 argument constructor it is whatever the caller passed, like `CustomRetrieval` in `student_tests.cpp`, which returns one result with score `99.0` that the real scoring can't produce. That's how I can tell the derived version ran. `RetrievalEngine::search` is marked `override`, so the compiler also checks that its signature matches the base.

If `search` were not `virtual`, the call would stick at compile time to `RetrievalStrategy::search`, because that is the static type of the pointer. The derived function would only hide the base one and `ProcessingCore` would never reach it. Here the base version is pure, so it couldn't be declared not virtual.

## 2. Ownership and lifetime - 1.5 points

Identify where one of the strategy objects is created, where ownership is transferred, and which object ultimately owns it. Explain how `std::unique_ptr` represents that ownership relationship and when the strategy object is destroyed.

Also explain why `ProcessingCore` is move-only and why the strategy base classes require virtual destructors.

I'll follow the chunker. The default constructor creates it with `std::make_unique<Chunker>(ChunkingPolicy{kMaxChunkTokens, kChunkOverlap, kParagraphPreferenceWindow})` and assigns it to `impl_->chunking_strategy`. In the 3 argument constructor the caller creates the object, passes it by value, and the constructor does `impl_->chunking_strategy = std::move(chunking_strategy)` after the null check. After it the caller's pointer is null, and a test checks that. The object ends up owned by `ProcessingCore::Impl`, which is owned by `impl_`. A `std::unique_ptr` says there is exactly one owner and it can't be copied, only moved. The chunker is destroyed when `ProcessingCore` is destroyed, because `~ProcessingCore() = default` destroys `impl_`, which destroys the three strategy pointers. It also happens when a move assignment replaces a core's old strategies. `student_tests.cpp` checks both with a `bool` flag the custom destructors set.

`ProcessingCore` is move only because it owns unique things. Copying `impl_` isn't possible, and copying the strategies would need a clone function the interfaces don't have, so the header deletes the copy operations and moving just hands over `impl_`.

## 3. Architecture, extensibility, and M1 compatibility - 1.5 points

Explain one specific architectural decision in your M2 implementation that makes the processing system extensible while preserving M1 behavior.

Identify the classes or interfaces involved and explain both:
- how the default configuration preserves M1 behavior; and
- how a different implementation can be substituted without changing the normal `ProcessingCore` API.

Include one plausible design alternative and explain why the M2 design is preferable for this milestone. The alternative does not need to be something you actually implemented.

The decision is that `ProcessingCore` only talks to chunking, retrieval and context through `ChunkingStrategy`, `RetrievalStrategy` and `ContextStrategy`, and gets them passed in. The default constructor builds `Chunker`, `RetrievalEngine` and `ContextBuilder`, which are the M1 classes. I didn't change their algorithms, I only made them derive from the interfaces. So a default `ProcessingCore` gives the same results as M1, and a test in `student_tests.cpp` checks that with a small document. To use something else, the caller passes different objects to the 3-argument constructor. `rebuild`, `search` and `build_context` only use the base types, so they don't change and neither does the public API. `rebuild` still builds its temporaries first and commits at the end, so if a strategy throws, the old corpus stays.

The alternative I thought about was an `enum` with an `if`/`switch` inside `ProcessingCore` to pick between built-in algorithms. It would work for the ones I already have, but every new one means editing `ProcessingCore`, and a test couldn't bring its own class, so I couldn't show runtime substitution. The interface version is better here because new behavior is just a new class.

## 4. Testing and defect reasoning - 1.5 points

Select one meaningful test from your `tests/student_tests.cpp`.

Explain:
- what M2 requirement the test validates;
- what specific implementation defect the test could detect; and
- why your test provides useful evidence beyond simply rerunning the supplied public tests.

If your test uses a custom strategy, explain how its observable behavior demonstrates that `ProcessingCore` is actually using runtime substitution.

I'm choosing the check "search forwards the query and k unchanged to the injected retrieval", from the first block of `student_tests.cpp`. It builds a core from `CustomChunker`, `CustomRetrieval` and `CustomContext`, which all share a `Probe`, calls `core.search` and checks that the `Probe` saw one call with the same query and `k`. It validates that `ProcessingCore::search` calls the injected `RetrievalStrategy` and passes the arguments through. The defect it can catch is a `search` that calls the strategy with a wrong or fixed `k`, or a changed query. I tried that in a scratch copy by hardcoding `k` in the call to `impl_->retrieval_strategy->search`. `public_tests` still passed, because its `FirstOnly` strategy ignores `k`, but my test failed. Replacing the call with a local `RetrievalEngine` is caught by both files, so what my test adds is checking the arguments.