## Specification

In short, Borsh is a non self-describing binary serialization format. It is designed to serialize any objects to canonical and deterministic set of bytes.

General principles:

- integers are little endian;
- sizes of dynamic containers are written before values as ~~`u32`~~ `Some integer type`;
- all unordered containers (hashmap/hashset) are ordered in lexicographic order by key (in tie breaker case on value);
- structs are serialized in the order of fields in the struct;
- enums are serialized with using ~~`u8`~~ `Some integer type` for the enum ordinal and then storing data inside the enum value (if present).

Formal specification:

<div>
  <table>
    <tr>
      <td>Informal type</td>
      <td><a href="https://doc.rust-lang.org/grammar.html">Rust EBNF </a> * </td>
      <td>Pseudocode</td>
    </tr>
    <tr>
      <td>Integers</td>
      <td>integer_type: ["u8" | "u16" | "u32" | "u64" | "u128" | "i8" | "i16" | "i32" | "i64" | "i128" ]</td>
      <td>little_endian(x)</td>
    </tr>
    <tr>
      <td>Floats</td>
      <td>float_type: ["f32" | "f64" ]</td>
      <td>
        err_if_nan(x)<br/>
        little_endian(x as integer_type)
      </td>
    </tr>
    <tr>
      <td>Unit</td>
      <td>unit_type: "()"</td>
      <td>We do not write anything</td>
    </tr>
    <tr>
      <td>Bool</td>
      <td>boolean_type: "bool"</td>
      <td>
        if x {<br/>
        &nbsp; repr(1 as u8)<br/>
        } else {<br/>
        &nbsp; repr(0 as u8)<br/>
        }
      </td>
    </tr>
    <tr>
      <td>Fixed sized arrays</td>
      <td>array_type: '[' ident ';' literal ']'</td>
      <td>
        for el in x {<br/>
        &nbsp; repr(el as ident)<br/>
        }
      </td>
    </tr>
    <tr>
      <td>Dynamic sized array</td>
      <td>vec_type: "Vec&lt;" ident '&gt;'</td>
      <td>
        repr(len() as <s>u32</s> some integer type)<br/>
        for el in x {<br/>
        &nbsp; repr(el as ident)<br/>
        }
      </td>
    </tr>
    <tr>
      <td>Struct</td>
      <td>struct_type: "struct" ident fields</td>
      <td>repr(fields)</td>
    </tr>
    <tr>
      <td>Fields</td>
      <td>fields: [named_fields | unnamed_fields]</td>
      <td></td>
    </tr>
    <tr>
      <td>Named fields</td>
      <td>named_fields: '{' ident_field0 ':' ident_type0 ',' ident_field1 ':' ident_type1 ',' ... '}'</td>
      <td>
        repr(ident_field0 as ident_type0)<br/>
        repr(ident_field1 as ident_type1)<br/>
        ...
      </td>
    </tr>
    <tr>
      <td>Unnamed fields</td>
      <td>unnamed_fields: '(' ident_type0 ',' ident_type1 ',' ... ')'</td>
      <td>
        repr(x.0 as type0)<br/>
        repr(x.1 as type1)<br/>
        ...
      </td>
    </tr>
    <tr>
      <td>Enum</td>
      <td>
        enum: 'enum' ident '{' variant0 ',' variant1 ',' ... '}'<br/>
        variant: ident [ fields ] ?
      </td>
      <td>
        Suppose X is the number of the variant that the enum takes.<br/>
        repr(X as <s>u8</s> some integer type)<br/>
        repr(x.X as fieldsX)
      </td>
    </tr>
    <tr>
      <td>HashMap</td>
      <td>hashmap: "HashMap&lt;" ident0, ident1 "&gt;"</td>
      <td>
        repr(x.len() as <s>u32</s> some integer type)<br/>
        for (k, v) in x.sorted_by_key() {<br/>
        &nbsp; repr(k as ident0)<br/>
        &nbsp; repr(v as ident1)<br/>
        }
      </td>
    </tr>
    <tr>
      <td>HashSet</td>
      <td>hashset: "HashSet&lt;" ident "&gt;"</td>
      <td>
        repr(x.len() as <s>u32</s> some integer type)<br/>
        for el in x.sorted() {<br/>
        &nbsp; repr(el as ident)<br/>
        }
      </td>
    </tr>
    <tr>
      <td>Option</td>
      <td>option_type: "Option&lt;" ident '&gt;'</td>
      <td>
        if x.is_some() {<br/>
        &nbsp; repr(1 as u8)<br/>
        &nbsp; repr(x.unwrap() as ident <br/>
        } else {<br/>
        &nbsp; repr(0 as u8)<br/>
        }
      </td>
    </tr>
    <tr>
      <td>String</td>
      <td>string_type: "String"</td>
      <td>
        encoded = utf8_encoding(x) as Vec&lt;u8&gt;<br/>
        repr(encoded.len() as <s>u32</s> some integer type)<br/>
        repr(encoded as Vec&lt;u8&gt;)
      </td>
    </tr>
  </table>
</div>

Note:

- Some parts of Rust grammar are not yet formalized, like enums and variants. We backwards derive EBNF forms of Rust grammar from [syn types](https://github.com/dtolnay/syn);
- We had to extend repetitions of EBNF and instead of defining them as `[ ident_field ':' ident_type ',' ] *` we define them as `ident_field0 ':' ident_type0 ',' ident_field1 ':' ident_type1 ',' ...` so that we can refer to individual elements in the pseudocode;
- We use `repr()` function to denote that we are writing the representation of the given element into an imaginary buffer.
