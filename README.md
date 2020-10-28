# grasp

Grep Clojure code using clojure.spec regexes. Inspired by [grape](https://github.com/bfontaine/grape).

## API

The `grasp.api` namespace currently exposes:

- `(grasp path-or-paths spec)`: returns matched sexprs in path or paths for
  spec. Accept source file, directory, jar file or classpath as string as well
  as a collection of strings for passing multiple paths. In case of a directory,
  it will be scanned recursively for source files ending with `.clj`, `.cljs` or
  `.cljc`.
- `(grasp-string string spec)`: returns matched sexprs in string for spec.
- `resolve-symbol`: returns the resolved symbol for a symbol, taking into
  account aliases and refers.

## Status

Very alpha. API will almost certainly change.

## Example usage

Assuming you have the following requires:

``` clojure
(require '[clojure.java.io :as io]
         '[clojure.pprint :as pprint]
         '[clojure.string :as str]
         '[clojure.spec.alpha :as s]
         '[grasp.api :as grasp :refer [grasp]])
```

### Find reify usages

Find `reify` usage with more than one interface:

``` clojure
(def clojure-core (slurp (io/resource "clojure/core.clj")))

(s/def ::clause (s/cat :sym symbol? :lists (s/+ list?)))

(s/def ::reify
  (s/cat :reify #{'reify}
         :clauses (s/cat :clause ::clause :clauses (s/+ ::clause))))

(def matches (grasp/grasp-string clojure-core ::reify))

(doseq [m matches]
  (prn (meta m))
  (pprint/pprint m)
  (println))
```

This outputs:

``` clojure
{:line 6974, :column 5, :end-line 6988, :end-column 56}
(reify
 clojure.lang.IDeref
 (deref [_] (deref-future fut))
 clojure.lang.IBlockingDeref
 (deref
  [_ timeout-ms timeout-val]
  (deref-future fut timeout-ms timeout-val))
 ...)

{:line 7107, :column 5, :end-line 7125, :end-column 16}
(reify
 clojure.lang.IDeref
 ...)
```
(output abbreviated for readability)

### Find usages based on resolved symbol

Find all usages of `clojure.set/difference`:

``` clojure
(defn table-row [sexpr]
  (-> (meta sexpr)
      (select-keys [:file :line :column])
      (assoc :sexpr sexpr)))

(->>
   (grasp/grasp "/Users/borkdude/git/clojure/src"
                (fn [sym]
                  (when (symbol? sym)
                    (= 'clojure.set/difference (grasp/resolve-symbol sym)))))
   (map table-row)
   pprint/print-table)
```

This outputs:

``` clojure
|                                                   :file | :line | :column |         :sexpr |
|---------------------------------------------------------+-------+---------+----------------|
|     /Users/borkdude/git/clojure/src/clj/clojure/set.clj |    49 |       7 |     difference |
|     /Users/borkdude/git/clojure/src/clj/clojure/set.clj |    62 |      14 |     difference |
|     /Users/borkdude/git/clojure/src/clj/clojure/set.clj |   172 |       2 |     difference |
|    /Users/borkdude/git/clojure/src/clj/clojure/data.clj |   112 |      19 | set/difference |
|    /Users/borkdude/git/clojure/src/clj/clojure/data.clj |   113 |      19 | set/difference |
| /Users/borkdude/git/clojure/src/clj/clojure/reflect.clj |   107 |      37 | set/difference |
```

### Grasp a classpath

Grasp the entire classpath for usage of `frequencies`:

``` clojure
(->> (grasp (System/getProperty "java.class.path") #{'frequencies})
     (take 2)
     (map (comp #(select-keys % [:file :line]) meta)))
```

Output:

``` clojure
({:file "sci/impl/namespaces.cljc", :line 815}
 {:file "sci/impl/namespaces.cljc", :line 815})
```

### Find defn with largest arg vector

``` clojure
(s/def ::args+body (s/cat :arg-vec vector? :exprs (s/+ seq?)))
(s/def ::fn-body (s/alt :single ::args+body :multi (s/+ (s/spec ::args+body))))
(s/def ::defn (s/cat :defn #{'defn} :name symbol? :some-stuff (s/* any?) :fn-body ::fn-body))

(defn arg-vecs [fn-body]
  (case (first fn-body)
    :single [(:arg-vec (second fn-body))]
    :multi (mapv :arg-vec (second fn-body))))

(s/def ::defn-with-large-arg-vec
  (s/and ::defn (fn [m]
                  (let [avs (arg-vecs (:fn-body m))]
                    (some #(> (count %) 7) avs)))))

(def matches (grasp (System/getProperty "java.class.path") ::defn-with-large-arg-vec))

(defn table-row [sexpr]
  (let [conformed (s/conform ::defn sexpr)
        m (meta sexpr)
        m (select-keys m [:file :line :column])]
    (assoc m :name (:name conformed))))

(pprint/print-table (map table-row matches))
```

This outputs:

``` clojure
|            :file | :line | :column |  :name |
|------------------+-------+---------+--------|
| clojure/core.clj |   353 |       1 | vector |
| clojure/core.clj |  6188 |       1 | update |
```

## License

Copyright © 2020 Michiel Borkent

Distributed under the EPL License. See LICENSE.
