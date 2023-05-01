# Pandas Style Guide

This guide offers recommendations for writing high-quality code with Pandas and DataFrames. It is intended to promote consistency, reliability, maintainability, and readability among practitioners, especially for production systems rather than ad-hoc exploratory work.

The need for this guide arises from the diverse backgrounds and language experiences of Pandas users, such as data scientists, data engineers, researchers, and software engineers, which can result in inconsistent coding styles and the use of poor practices.



## Column selection

```python
# Good
df['column']

# Bad
df.column
```

When selecting a column in a dataframe users should use dictionary like selection e.g. `df['column']` and not property selection e.g. `df.column`.

Why:
* It makes it more explict to the user that you are accessing a column and not a standard property or method
* Not all column names can be represented as a property - it must be a valid Python variable name. The column name also must not clash with an existing method or property

## Copy vs re-assignment

```python
# Good
df = df.drop(columns='A')

# Bad
df.drop(columns='A', inplace=True)
```

Some operations can either be performed inplace or by re-assignment. Users should always opt to re-assignment for better readability. Both operation, with a few exceptions, perform a data copy and therefore don't have any performance differences.

## Querying

```python
# Good
mask = df.loc[df.col > 3, :]

# Bad
mask = df[df.col > 3]
```

When querying a dataframe users should use `.loc` and not the shorthand `[]` notation.

Why:
* This avoids ambiguity with the `[]` operator which can be used for both column selection and querying
* SettingWithCopy errors can occur when using the `[]` operator 

## Schema contract

```python
# Good (file input)
df = pd.read_csv('data.csv')
df = df[['col1', 'col2', 'col3']]

# Good (collection input)
df = pd.DataFrame.from_records(list_of_dicts)
df = df[['col1', 'col2', 'col3']]
```

After a DataFrame initialisation, users should explicitly select the columns to use, even if all are selected. This creates a clear contract between the code and user about what is expected in the downstream data schema.

It is also recommended to explicitly specify data types as well. When reading files data types can subtly change with small alterations e.g. from integer to float, or numerical to string. It is helpful for the program to throw an error if the input is unexpected, otherwise it will continue silently.

```python
# Good
df = pd.read_csv('data.csv', dtype={'col1': 'str', 'col2': 'int', 'col3': 'float'})
df = df[['col1', 'col2', 'col3']]
```

## Merge contract (join)

```python
# Good
df_3 = df_1.merge(
    df_2,
    how='inner',
    on='col1',
    validate='1:1'
)

# Bad
df_3 = df_1.merge(df_2)
```

Users should explicitly specify the `how` and `on` parameters of a merge for readability, even if the default parameters would produce the same result.

It is also important to validate the merge type using the `validate` parameter, this prevents unexpected "merge duplication". Upstream data can subtly changed which produces multiple rows per merge key. If no validation is performed, the data here will silently multiply and duplicate, as the operation will be promoted to a "many-to-one" or "many-to-many" producing "merge duplication". It is therefore useful to have the merge assumptions explicitly stated.

Also, don't deduplicate rows after a merge to remove merge duplication. Remove duplicates before joining, or even better determine why there are unexpected duplicate keys and remove them upstream. Merge duplication are computationally and memory expensive and produce hard to debug data bugs.

## Empty columns

```python
# Good
df['new_col_float'] = np.nan
df['new_col_int'] = pd.Series(dtype='int')
df['new_col_str'] = pd.Series(dtype='object')

# Bad
df['new_col_int'] = 0
df['new_col_str'] = ''
```

If a new empty column is needed always use NaN values. Never use "filler" values such as zeros or empty strings. This preserves the ability to use methods such as `isnull` or `notnull`.

## Mutability of DataFrames

```python
# Bad
def func_1(df: pd.DataFrame) -> pd.DataFrame:
    df['new_col'] = df['col1'] + 1
    return df

df = func_1(df)
```

In large code bases, it can be tempting to keep adding columns to a dataframe and use it as the data exchange format between methods and functions. This however leads to code which is difficult to maintain as readers can't directly determine the schema of a DataFrame without running the code. It is more preferable to use dataclasses as an interchange format as the class definition is effectively immutable and specified explicitly (or if Python < 3.6 use namedtuples).

## Spaghetti Code (Bad)

```python
# Import data
df_raw = pd.read_csv("path/to/data/file.csv")

# Clean data
df_raw["id"] = df_raw["id"].astype(str)
df_merged = df_raw.merge(df2, on="id")
df_final = df_merged.drop(columns=["col_5", "col_6", "col_7"])

# Investigate data
df_agg = df_final.groupby("id").size()
```
Above is an example of spaghetti code. In each step of our data cleaning process, we create a new dataframe. This ultamitely clutters our namespace, uses up memory, and is difficult to understand

## Method Chaining
```python
new_df = (                          # Wrap everything in ()'s
    original_df                     # Name of data frame to modify
    .query("text_length > 140")     # Subset based on text length
    .sort_values(by="text_length")  # Sort entire df by text length
    .reset_index()                  # Reset index of subsetted df
)
```
Method chaining is the process of daisy chaining our Pandas methods together to acheive our desired dataframe state. This allows us to keep our namespace clean, reduces memory usage, and is much more human readable

## Assign Method
```python

df= df[["col1", "col2", "col3"]]

df = df.assign(                          
     col1= df["col1"].astype(str)
     col2 = df["col2"].apply(lambda x: x*100)
     col4 = df["col3"].apply(pd.round(3))             
)
```

Pandas `.assign()` method allows you to apply multiple functions to columns in a dataframe sequentially. Notice how we updated `col1` and `col2` as well as created a new column `col4`

