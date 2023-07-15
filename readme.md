# REZUSTAND

[![Build Status](https://img.shields.io/github/actions/workflow/status/lonelyhentai/rezustand/lint-and-type.yml?branch=main&style=flat)](https://github.com/lonelyhentai/rezustand/actions?query=workflow%3ALint)
[![Build Size](https://img.shields.io/bundlephobia/minzip/rezustand?label=bundle%20size&style=flat)](https://bundlephobia.com/result?p=rezustand)
[![Version](https://img.shields.io/npm/v/rezustand?style=flat)](https://www.npmjs.com/package/rezustand)

A toolset making zustand easy to use!

## Installation

```shell
npm install rezustand
```

## Example

In `appStore.ts`:

```typescript
import {
  easyCreator,
  easyNSSliceCreator,
  easyStoreContext,
  storeApiSubscribeWithSelector
} from "./rezustand";
import type {
  EasyInferStoreTypes,
  EasyInferNSSliceStateType,
  EasyInferCreateSliceStateType
} from "./rezustand";
import { createStore } from "zustand";
import { immer } from "zustand/middleware/immer";

const InitProps = {
  a: 1,
  b: 2
};

const createAppSlice = () => {
  return easyCreator({
    setup() {
      return {
        ...InitProps,
        haha: () => {
          const api = this.api;
          const state = api.getState();
          console.log(`haha + ${state.a}`);
        },
        incA: () => {
          const { a } = this.get();

          this.set((s) => void (s.a = a + 1));
        },
        getInitialProps: async () => {
          this.methods.incA();
        }
      };
    }
  });
};

const createListSlice = <NS extends string>(
  ns: NS,
  initProps: typeof InitProps
) => {
  return easyNSSliceCreator({
    ns,
    ext: initProps,
    setup() {
      return {
        listData: [] as number[],
        getListData: () => {
          const prevS = this.get();
          const prevList = prevS.listData;
          this.set((s) => {
            s.listData = [...prevList, prevS.ext.a];
          });
        },
        subscribeListChange: () => {
          storeApiSubscribeWithSelector(
            this.api,
            (s) => s.listData,
            (listData) => {
              console.log(`${this.ns} list data changed`, listData);
            },
            {
              fireImmediately: true
            }
          );
        }
      };
    }
  });
};

type AppState = EasyInferCreateSliceStateType<typeof createAppSlice> &
  EasyInferNSSliceStateType<
    "listA",
    EasyInferCreateSliceStateType<typeof createListSlice>
  >;

const createAppStore = () =>
  createStore<AppState>()(
    immer((set, get, api) => ({
      ...createAppSlice()(set, get, api),
      ...createListSlice("listA", InitProps)(set, get, api)
    }))
  );

export type AppStoreTypes = EasyInferStoreTypes<
  ReturnType<typeof createAppStore>
>;

const {
  useContextStore: useAppStore,
  StoreContext: AppStoreContext,
  StoreProvider: AppStoreProvider
} = easyStoreContext(createAppStore);

export { useAppStore, createAppStore, AppStoreContext, AppStoreProvider };
```

In `App.tsx`:

```tsx
import "./styles.css";
import { AppStoreProvider, useAppStore } from "./appStore";
import { useEffect } from "react";

function ChildView() {
  const { a, incA, subscribeListChange, getListData } = useAppStore((s) => ({
    a: s.a,
    incA: s.incA,
    subscribeListChange: s.listA.subscribeListChange,
    getListData: s.listA.getListData
  }));

  useEffect(() => {
    const unsub = subscribeListChange();
    getListData();
    return unsub;
  }, [subscribeListChange, getListData]);

  return (
    <>
      <h1>Hello CodeSandbox</h1>
      <h2>a: {a}</h2>
      <button onClick={() => incA()}>add a</button>
    </>
  );
}

export default function App() {
  return (
    <AppStoreProvider>
      <div className="App">
        <ChildView />
      </div>
    </AppStoreProvider>
  );
}
```
