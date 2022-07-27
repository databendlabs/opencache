all: test lint fmt doc

test: lint fmt
	cargo test

fmt:
	cargo fmt

fix:
	cargo fix --allow-staged

doc:
	cargo doc --all --no-deps

lint:
	cargo fmt
	cargo clippy --all-targets -- -D warnings -A clippy::bool-assert-comparison

clean:
	cargo clean

.PHONY: test fmt lint clean doc
