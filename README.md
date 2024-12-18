# `capacity_builder`

Builders where the code to calculate the capacity is the same as the code to
write what's being built.

## Overview

Sometimes you have some complex code that would be a bit of a pain to calculate
the capacity of or could risk easily getting out of sync with the
implementation. This crate makes keeping it in sync easier because it's the same
code.

```rs
use capacity_builder::StringBuilder;

let text = StringBuilder::build(|builder| {
  for (i, import_module) in import_modules.iter().enumerate() {
    builder.append("// ");
    builder.append(i);
    builder.append(" import\n");
    builder.append("import \"");
    builder.append(import_module);
    builder.append("\";\n");
  }
})?;
```

Behind the scenes it runs the closure once to compute the capacity and a second
time to write the string.

## Implementing faster `.to_string()` and `std::fmt::Display`

The default `.to_string()` implementation reuses `std::fmt::Display`. This is
slow because the capacity isn't set.

This crate provides a `StringBuildable` trait and `#[derive(FastDisplay)]` macro
for implementing `.to_string()` and `std::fmt::Display` using this crate.

```rs
use capacity_builder::FastDisplay;
use capacity_builder::StringBuildable;
use capacity_builder::StringBuilder;

#[derive(Debug, FastDisplay)]
pub struct Version {
  // ...
}

impl StringBuildable for Version {
  fn string_build_with<'a>(
    &'a self,
    builder: &mut capacity_builder::StringBuilder<'a, '_, '_>,
  ) {
    builder.append(version.major);
    builder.append('.');
    builder.append(version.minor);
    builder.append('.');
    builder.append(version.patch);
    if !version.pre.is_empty() {
      builder.append('-');
      for (i, part) in version.pre.iter().enumerate() {
        if i > 0 {
          builder.append('.');
        }
        builder.append(part);
      }
    }
    if !version.build.is_empty() {
      builder.append('+');
      for (i, part) in version.build.iter().enumerate() {
        if i > 0 {
          builder.append('.');
        }
        builder.append(part);
      }
    }
  }
}
```

Now `version.to_string()` will be fast and return a string that has an accurate
capacity.

Additionally, this type can now be appended to other builders:

```rs
builder.append(&version);
```

Side note: You may have noticed that no errors are necessary to surface. This is
because errors when formatting are really rare and if an error is encountered it
will store it to surface at the end and the rest of the `append` statements stop
formatting.

## `BytesBuilder`

The bytes builder is similar to the `StringBuilder`:

```rs
use capacity_builder::BytesBuilder;

let bytes = BytesBuilder::build(|builder| {
  builder.append_le(123);
  builder.append("example");
  builder.append(other_bytes);
})?;
```

You can implement `BytesAppendable` to allow appending a struct to a builder.

```rs
struct MyStruct;

impl<'a> BytesAppendable<'a> for &'a MyStruct {
  fn append_to_builder(self, builder: &mut BytesBuilder<'a, '_>) {
    builder.append("Hello");
    builder.append(" there!");
  }
}

let bytes = BytesBuilder::build(|builder| {
  builder.append(&MyStruct);
})
.unwrap();
assert_eq!(bytes, b"Hello there!");
```

## Features

1. The builder prevents adding owned data—only references.
   - This helps to prevent accidentally allocating data multiple times in the
     closure.
1. Errors when capacity cannot be reserved.
1. For the string builder, types other than references can be provided.
   - Numbers get written with the [itoa](https://crates.io/crates/itoa) crate.

## Tips

- Do any necessary allocations before running the closure.
- Measure before and after using this crate to ensure you're not slower.
