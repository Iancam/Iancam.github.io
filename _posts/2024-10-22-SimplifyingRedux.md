# Simplifying Redux
Redux is a popular state management library for React, but its implementation can be unnecessarily complex, requiring multiple files to handle a single action-reducer. To simplify this, I propose a library that handles state declaratively, exposing an object with all actions organized by resource name.

## Motivation
In my experience, reducers and their corresponding actions often follow the same logic: setting and updating objects and literals. MOst other state manipulations could either be simpled into these two operations, or moved elsewhere in the redux flow. By applying this rule, I simplified my redux implementation significantly. 

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

const {reducers:reducer1, actions:actions1} = fromStateMap(exampleStateMap);

const reducers = [reducer1]; // include reducers that don't follow the statemap pattern here

const reduced = listOfReducersToReducer(reducers);
export const actions = actions1;

export default combineReducers(reduced);

// example action

actions.setBooks({bookId1:{id: "bookId1", title:"The Great Gatsby"}})

```

## The code 

__utils.js__
``` js
import { mapValues, toPairs } from "lodash";

/**
 * Takes an array of reducers, and returns a single reducer that applies all of
 * the reducers in the array in sequence. The result of each reducer is passed
 * as the state to the next reducer.
 *
 * @param  {Array<Function>} reducerArray - Array of reducer functions to apply
 *                                          in sequence.
 * @return {Function}                     - A single reducer that applies all of
 *                                          the reducers in the array in sequence.
 */
function reduceReducers(reducerArray) {
  return (previous, current) =>
    reducerArray.reduce((p, r) => r(p, current), previous);
}

/**
 * Takes a list of objects with reducer functions as values, and returns a single
 * object with the same keys, but with the reducer functions combined into a
 * single reducer using the reduceReducers function.
 *
 * @param {Object[]} reducers - the list of objects with reducer functions
 * @returns {Object} the combined reducer object
 */
export const listOfReducersToReducer = (reducers) => {
  // Maps each reducer object to an array of key-value pairs using toPairs,
  // then flattens the resulting arrays into a single array of key, reducer pairs.
  const flattened = reducers
    .map((reducerObj) => toPairs(reducerObj))
    .reduce((acc, array) => [...acc, ...array]);

  // Groups the flattened pairs by key, creating an object where each key 
  // has an array of corresponding reducers.
  const grouped = flattened.reduce((acc, pair) => {
    const key = pair[0];
    const val = pair[1];
    let array = (acc[key] && [...acc[key]]) || [];
    acc[key] = [...array, val];
    return acc;
  }, {});
  // transform each array of reducers into a single reducer using the reduceReducers function.
  return mapValues(grouped, (val) => reduceReducers(val));
};
```

__resourcesToRedux.js__
``` js

// getters, and setters/reducers

import { camelCase, keyBy, keys, mapValues, snakeCase } from "lodash";


// These could be extended to handle lists and object literals
const baseReducers = {
  set: (defaultState) => (state = defaultState, action) => action.payload,
  update: (defaultState) => (state = defaultState, action) => ({
    ...state,
    ...action.payload,
  }),
};

const action = (type) => (payload) => ({ type, payload });

/**
 *
 * generates Redux actions and reducers from a given state map object.
 * It takes an object with state names as keys and default values as values, 
 * and returns an object with corresponding actions and reducers for each state. 
 * The actions include set and update functions for each state, and the 
 * reducers handle these actions to update the state accordingly.
 * 
 * For example, from {cat: "meow"} you get actions:
 *  - setCat(),
 *  - updateCat()
 * and reducers for types "SET_CAT", and "UPDATE_CAT"
 * 
 * For simplicity, this library expects that all states are objects.
 * To handle a list, index items on an ID field, or create an id from the list index.
 *
 *
 */
export function fromStateMap(stateMap) {
  const resourceNames = Object.keys(stateMap).filter(
    (k) => typeof stateMap[k] !== "function"
  );
  const prefixes = "update set".split(" ");

  const reducers = resourceNames.reduce((reducers, name) => {
    const defaultState = stateMap[name];
    const handlers = toHandlers(prefixes, name, defaultState);

    const reducer = (state = defaultState, action) => {
      return handlers[action.actiontype]
        ? handlers[action.actiontype](state, action)
        : state;
    };
    return {
      ...reducers,
      [name]: reducer,
    };
  }, {});

  const actionTypes = resourceNames
    .map((name) =>
      prefixes.map((prefix) => ({
        prefix,
        type: snakeCase(`${prefix}_${name}`),
      }))
    )
    .flat();

  const actions = actionTypes
    .map(({ type }) => ({
      fname: camelCase(type),
      action: action(type),
    }))
    .reduce((obj, curr) => ({ ...obj, [curr.fname]: curr.action }), {});

  return { actions, reducers };
}

/**
 * Generates Redux handlers for different prefixes based on the state name and
 * default state.
 */
function toHandlers(prefixes, name, defaultState) {
  return prefixes
    .map((prefix) => ({ prefix, type: snakeCase(`${prefix}_${name}`) }))
    .reduce(
      (all, curr) => ({
        ...all,
        [curr.type.toUpperCase()]: baseReducers[curr.prefix](defaultState),
      }),
      {}
    );
}

console.log(fromStateMap({ catsAndMonkeys: [], dogs: {} }).reducers.dogs);

```

