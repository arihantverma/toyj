- #+BEGIN_QUERY
  {:title "All Cards"
   :query [:find (pull ?b [*])
           :where
           [?b :block/refs ?p]
           [?p :block/name "card"]]}
  #+END_QUERY