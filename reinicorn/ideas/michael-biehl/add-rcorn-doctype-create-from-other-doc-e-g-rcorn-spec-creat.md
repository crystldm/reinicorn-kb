# Add 'rcorn <doctype> create from <other-doc>' e.g. 'rcorn spec create from <idea

**Date:** 2026-07-31
**Author:** Michael Biehl
**Status:** new

## Description

Add 'rcorn <doctype> create from <other-doc>' e.g. 'rcorn spec create from <idea-slug>': graduate a kb doc into another type, carrying over its content/context as the starting point and linking back to the source doc.

## Notes

This melds perfectly with `rcorn plan create from <spec>` — the same graduation mechanism covers the whole pipeline: idea → spec → plan. Each step seeds the new doc from its source and links back, so `create from` becomes the uniform way a doc advances to the next stage.
