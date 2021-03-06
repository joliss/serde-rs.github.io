# Manually implementing Deserialize for a struct

Only when [codegen](codegen.md) is not getting the job done.

The `Deserialize` impl below corresponds to the following struct:

```rust
struct Duration {
    secs: u64,
    nanos: u32,
}
```

Deserializing a struct is somewhat more complicated than [deserializing a
map](deserialize-map.md) in order to avoid allocating a String to hold the field
names. Instead there is a `Field` enum which is deserialized from a `&str`.

The implementation supports two possible ways that a struct may be represented
by a data format: as a seq like in Bincode, and as a map like in JSON.

```rust
impl Deserialize for Duration {
    fn deserialize<D>(deserializer: D) -> Result<Self, D::Error>
        where D: Deserializer
    {
        enum Field { Secs, Nanos };

        impl Deserialize for Field {
            fn deserialize<D>(deserializer: D) -> Result<Field, D::Error>
                where D: Deserializer
            {
                struct FieldVisitor;

                impl Visitor for FieldVisitor {
                    type Value = Field;

                    fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
                        formatter.write_str("`secs` or `nanos`")
                    }

                    fn visit_str<E>(self, value: &str) -> Result<Field, E>
                        where E: Error
                    {
                        match value {
                            "secs" => Ok(Field::Secs),
                            "nanos" => Ok(Field::Nanos),
                            _ => Err(Error::unknown_field(value, FIELDS)),
                        }
                    }
                }

                deserializer.deserialize_struct_field(FieldVisitor)
            }
        }

        struct DurationVisitor;

        impl Visitor for DurationVisitor {
            type Value = Duration;

            fn expecting(&self, formatter: &mut fmt::Formatter) -> fmt::Result {
                formatter.write_str("struct Duration")
            }

            fn visit_seq<V>(self, mut visitor: V) -> Result<Duration, V::Error>
                where V: SeqVisitor
            {
                let secs: u64 = match visitor.visit()? {
                    Some(value) => value,
                    None => {
                        return Err(Error::invalid_length(0, &self));
                    }
                };
                let nanos: u32 = match visitor.visit()? {
                    Some(value) => value,
                    None => {
                        return Err(Error::invalid_length(1, &self));
                    }
                };
                Ok(Duration::new(secs, nanos))
            }

            fn visit_map<V>(self, mut visitor: V) -> Result<Duration, V::Error>
                where V: MapVisitor
            {
                let mut secs = None;
                let mut nanos = None;
                while let Some(key) = visitor.visit_key()? {
                    match key {
                        Field::Secs => {
                            if secs.is_some() {
                                return Err(Error::duplicate_field("secs"));
                            }
                            secs = Some(visitor.visit_value()?);
                        }
                        Field::Nanos => {
                            if nanos.is_some() {
                                return Err(Error::duplicate_field("nanos"));
                            }
                            nanos = Some(visitor.visit_value()?);
                        }
                    }
                }
                let secs = match secs {
                    Some(secs) => secs,
                    None => return Err(Error::missing_field("secs")),
                };
                let nanos = match nanos {
                    Some(nanos) => nanos,
                    None => return Err(Error::missing_field("nanos")),
                };
                Ok(Duration::new(secs, nanos))
            }
        }

        const FIELDS: &'static [&'static str] = &["secs", "nanos"];
        deserializer.deserialize_struct("Duration", FIELDS, DurationVisitor)
    }
}
```
