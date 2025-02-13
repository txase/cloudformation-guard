# Known Issues and Workarounds

No software is perfect, here is a list of known issues /gotchas that is worth noting with potential workarounds when you do encounter them (and for some, where you don’t really have a solution)

1. Queries assigned to variables (see [Guard: Variable, Projections and Interpolations](QUERY_PROJECTION_AND_INTERPOLATION.md)) can be accessed using two forms when defining clauses, E.g. `let api_gws = Resources.*[ Type == 'AWS::ApiGateway::RestApi' ]`

```
%api_gws.Properties.EndpointConfiguration.Types[*] == "PRIVATE"`
```

or

```
%api_gws {
    Properties.EndpointConfiguration.Types[*] == "PRIVATE"
}
```

The block form iterates over all `AWS::ApiGateway::RestApi` resources found in the input. The first form short circuits and returns immediately after the first resource failure.

> **Workaround**: use the block form to traverse all values to show all resource failures and not just the first one that failed. We are tracking to resolve this issue. 2. Need `when` guards with filter expressions- When a query uses filters like `Resources.*[ Type == 'AWS::ApiGateway::RestApi' ]`, if there are no `ApiGatway` resources, then Guard will fail the clause today when performing the check

```
%api_gws.Properties.EndpointConfiguration.Types[*] == "PRIVATE"
```

> **Workaround**: assign filters to variables and use `when` condition check e.g.

```
let api_gws = Resources.*[ Type == 'AWS::ApiGateway::RestApi' ]
    when %api_gws !empty { ...}
```

3. When performing `!=` comparison, if the values are incompatible like comparing a `string` to `int`, an error is thrown internally but currently suppressed and converted to `false` to satisfy the requirements of Rust’s [PartialEq](https://doc.rust-lang.org/std/cmp/trait.PartialEq.html). We are tracking to release a fix for this issue soon.
4. `exists` and `empty` checks do not display the JSON pointer path inside the document in the error messages. Both these clauses often have retrieval errors which does not maintain this traversal information today. We are tracking to resolve this issue.
5. <a name="function-limitation"></a> **No support for inline functions**

   We **do not** support inline usage of functions at the moment. The support for built-in functions is currently limited to assignment of the return value to a variable.

   Consider an example wherein our template has a node named `Instances` which is a collection. We need to author a rule that checks to ensure this collection contains a certain number of minimum items, say 2.

   This is currently **NOT SUPPORTED**:

   ```
   # Not supported at the moment

   rule INSTANCES_COUNT_CHECK {
      count(Instances.*) < 2
      << Violation: We should have at least 2 instances >>
   }
   ```

   While the above code snippet might be tempting to use as it's more intuitive, we haven't made the changes required to support it in our grammar yet.

   > **Workaround**: Assign function value to a variable and then use this variable thereafter in all clauses that follow including the conditions.

   So, our example rule now becomes:

   ```
   # Use this instead

   rule INSTANCES_COUNT_CHECK {
      let no_of_instances = count(Instances.*)

      %no_of_instances < 2
      << Violation: We should have at least 2 instances >>
   }
   ```
