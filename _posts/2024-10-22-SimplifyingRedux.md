---
layout: default
title: "Simplfying Redux"
date: 2024-10-22
author: Ian Campbell
---

# Simplifying Redux

Redux is a popular state management library for React, but its implementation can be unnecessarily complex, requiring multiple files to handle a single action-reducer. To simplify this, I created a library that handles state declaratively, generating an object with all actions organized by resource name.

## Motivation

In my experience, reducers and their corresponding actions often follow the same logic: setting and updating objects and literals. Most other state manipulations could either be simplified into these two operations, or moved elsewhere in the redux flow. By applying this rule, I simplified my redux implementation significantly.

It also created an opportunity: since these reducers and actions shared the same structure, except their resource name, we could generate them from a list of resource names. This would save loads of code and complexity.

## Example Usage

```js
import { combineReducers } from "redux";
import { fromStateMap } from "./stateToRedux";
import { listOfReducersToReducer } from "./utils";
//import reducers here

const exampleStateMap = {
  books: {},
  currentBook: null,
};

const { reducers: reducer1, actions: actions1 } = fromStateMap(exampleStateMap);

const reducers = [reducer1]; // include reducers that don't follow the statemap pattern here

const reduced = listOfReducersToReducer(reducers);
export const actions = actions1;

export default combineReducers(reduced);

// example action

actions.setBooks({ bookId1: { id: "bookId1", title: "The Great Gatsby" } });
```
