## Description

该项目基于 [go-ozzo/ozzo-validation](https://github.com/go-ozzo/ozzo-validation) 稍做修改

## Requirements

Go 1.13 or above.


## Getting Started

The ozzo-validation package mainly includes a set of validation rules and two validation methods. You use 
validation rules to describe how a value should be considered valid, and you call either `valid.Validate()`
or `valid.ValidateStruct()` to validate the value.


### Installation

Run the following command to install the package:

```
go get github.com/maksliu/valid
```

### Validating a Simple Value

For a simple value, such as a string or an integer, you may use `valid.Validate()` to validate it. For example, 

```go
package main

import (
	"fmt"

	"github.com/maksliu/valid"
	"github.com/maksliu/valid/is"
)

func main() {
	data := "example"
	err := valid.Validate(data,
		valid.Required,       // not empty
		valid.Length(5, 100), // length between 5 and 100
		is.URL,                    // is a valid URL
	)
	fmt.Println(err)
	// Output:
	// must be a valid URL
}
```

The method `valid.Validate()` will run through the rules in the order that they are listed. If a rule fails
the validation, the method will return the corresponding error and skip the rest of the rules. The method will
return nil if the value passes all validation rules.


### Validating a Struct

For a struct value, you usually want to check if its fields are valid. For example, in a RESTful application, you
may unmarshal the request payload into a struct and then validate the struct fields. If one or multiple fields
are invalid, you may want to get an error describing which fields are invalid. You can use `valid.ValidateStruct()`
to achieve this purpose. A single struct can have rules for multiple fields, and a field can be associated with multiple 
rules. For example,

```go
type Address struct {
	Street string
	City   string
	State  string
	Zip    string
}

func (a Address) Validate() error {
	return valid.ValidateStruct(&a,
		// Street cannot be empty, and the length must between 5 and 50
		valid.Field(&a.Street, valid.Required, valid.Length(5, 50)),
		// City cannot be empty, and the length must between 5 and 50
		valid.Field(&a.City, valid.Required, valid.Length(5, 50)),
		// State cannot be empty, and must be a string consisting of two letters in upper case
		valid.Field(&a.State, valid.Required, valid.Match(regexp.MustCompile("^[A-Z]{2}$"))),
		// State cannot be empty, and must be a string consisting of five digits
		valid.Field(&a.Zip, valid.Required, valid.Match(regexp.MustCompile("^[0-9]{5}$"))),
	)
}

a := Address{
    Street: "123",
    City:   "Unknown",
    State:  "Virginia",
    Zip:    "12345",
}

err := a.Validate()
fmt.Println(err)
// Output:
// Street: the length must be between 5 and 50; State: must be in a valid format.
```

Note that when calling `valid.ValidateStruct` to validate a struct, you should pass to the method a pointer 
to the struct instead of the struct itself. Similarly, when calling `valid.Field` to specify the rules
for a struct field, you should use a pointer to the struct field. 

When the struct validation is performed, the fields are validated in the order they are specified in `ValidateStruct`. 
And when each field is validated, its rules are also evaluated in the order they are associated with the field.
If a rule fails, an error is recorded for that field, and the validation will continue with the next field.


### Validating a Map

Sometimes you might need to work with dynamic data stored in maps rather than a typed model. You can use `valid.Map()`
in this situation. A single map can have rules for multiple keys, and a key can be associated with multiple 
rules. For example,

```go
c := map[string]interface{}{
	"Name":  "Qiang Xue",
	"Email": "q",
	"Address": map[string]interface{}{
		"Street": "123",
		"City":   "Unknown",
		"State":  "Virginia",
		"Zip":    "12345",
	},
}

err := valid.Validate(c,
	valid.Map(
		// Name cannot be empty, and the length must be between 5 and 20.
		valid.Key("Name", valid.Required, valid.Length(5, 20)),
		// Email cannot be empty and should be in a valid email format.
		valid.Key("Email", valid.Required, is.Email),
		// Validate Address using its own validation rules
		valid.Key("Address", valid.Map(
			// Street cannot be empty, and the length must between 5 and 50
			valid.Key("Street", valid.Required, valid.Length(5, 50)),
			// City cannot be empty, and the length must between 5 and 50
			valid.Key("City", valid.Required, valid.Length(5, 50)),
			// State cannot be empty, and must be a string consisting of two letters in upper case
			valid.Key("State", valid.Required, valid.Match(regexp.MustCompile("^[A-Z]{2}$"))),
			// State cannot be empty, and must be a string consisting of five digits
			valid.Key("Zip", valid.Required, valid.Match(regexp.MustCompile("^[0-9]{5}$"))),
		)),
	),
)
fmt.Println(err)
// Output:
// Address: (State: must be in a valid format; Street: the length must be between 5 and 50.); Email: must be a valid email address.
```

When the map validation is performed, the keys are validated in the order they are specified in `Map`. 
And when each key is validated, its rules are also evaluated in the order they are associated with the key.
If a rule fails, an error is recorded for that key, and the validation will continue with the next key.


### Validation Errors

The `valid.ValidateStruct` method returns validation errors found in struct fields in terms of `valid.Errors` 
which is a map of fields and their corresponding errors. Nil is returned if validation passes.

By default, `valid.Errors` uses the struct tags named `json` to determine what names should be used to 
represent the invalid fields. The type also implements the `json.Marshaler` interface so that it can be marshaled 
into a proper JSON object. For example,

```go
type Address struct {
	Street string `json:"street"`
	City   string `json:"city"`
	State  string `json:"state"`
	Zip    string `json:"zip"`
}

// ...perform validation here...

err := a.Validate()
b, _ := json.Marshal(err)
fmt.Println(string(b))
// Output:
// {"street":"the length must be between 5 and 50","state":"must be in a valid format"}
```

You may modify `valid.ErrorTag` to use a different struct tag name.

If you do not like the magic that `ValidateStruct` determines error keys based on struct field names or corresponding
tag values, you may use the following alternative approach:

```go
c := Customer{
	Name:  "Qiang Xue",
	Email: "q",
	Address: Address{
		State:  "Virginia",
	},
}

err := valid.Errors{
	"name": valid.Validate(c.Name, valid.Required, valid.Length(5, 20)),
	"email": valid.Validate(c.Name, valid.Required, is.Email),
	"zip": valid.Validate(c.Address.Zip, valid.Required, valid.Match(regexp.MustCompile("^[0-9]{5}$"))),
}.Filter()
fmt.Println(err)
// Output:
// email: must be a valid email address; zip: cannot be blank.
```

In the above example, we build a `valid.Errors` by a list of names and the corresponding validation results. 
At the end we call `Errors.Filter()` to remove from `Errors` all nils which correspond to those successful validation 
results. The method will return nil if `Errors` is empty.

The above approach is very flexible as it allows you to freely build up your validation error structure. You can use
it to validate both struct and non-struct values. Compared to using `ValidateStruct` to validate a struct, 
it has the drawback that you have to redundantly specify the error keys while `ValidateStruct` can automatically 
find them out.


### Internal Errors

Internal errors are different from validation errors in that internal errors are caused by malfunctioning code (e.g.
a validator making a remote call to validate some data when the remote service is down) rather
than the data being validated. When an internal error happens during data validation, you may allow the user to resubmit
the same data to perform validation again, hoping the program resumes functioning. On the other hand, if data validation
fails due to data error, the user should generally not resubmit the same data again.

To differentiate internal errors from validation errors, when an internal error occurs in a validator, wrap it
into `valid.InternalError` by calling `valid.NewInternalError()`. The user of the validator can then check
if a returned error is an internal error or not. For example,

```go
if err := a.Validate(); err != nil {
	if e, ok := err.(valid.InternalError); ok {
		// an internal error happened
		fmt.Println(e.InternalError())
	}
}
```


## Validatable Types

A type is validatable if it implements the `valid.Validatable` interface. 

When `valid.Validate` is used to validate a validatable value, if it does not find any error with the 
given validation rules, it will further call the value's `Validate()` method. 

Similarly, when `valid.ValidateStruct` is validating a struct field whose type is validatable, it will call 
the field's `Validate` method after it passes the listed rules.

> Note: When implementing `valid.Validatable`, do not call `valid.Validate()` to validate the value in its
> original type because this will cause infinite loops. For example, if you define a new type `MyString` as `string`
> and implement `valid.Validatable` for `MyString`, within the `Validate()` function you should cast the value 
> to `string` first before calling `valid.Validate()` to validate it.

In the following example, the `Address` field of `Customer` is validatable because `Address` implements 
`valid.Validatable`. Therefore, when validating a `Customer` struct with `valid.ValidateStruct`,
validation will "dive" into the `Address` field.

```go
type Customer struct {
	Name    string
	Gender  string
	Email   string
	Address Address
}

func (c Customer) Validate() error {
	return valid.ValidateStruct(&c,
		// Name cannot be empty, and the length must be between 5 and 20.
		valid.Field(&c.Name, valid.Required, valid.Length(5, 20)),
		// Gender is optional, and should be either "Female" or "Male".
		valid.Field(&c.Gender, valid.In("Female", "Male")),
		// Email cannot be empty and should be in a valid email format.
		valid.Field(&c.Email, valid.Required, is.Email),
		// Validate Address using its own validation rules
		valid.Field(&c.Address),
	)
}

c := Customer{
	Name:  "Qiang Xue",
	Email: "q",
	Address: Address{
		Street: "123 Main Street",
		City:   "Unknown",
		State:  "Virginia",
		Zip:    "12345",
	},
}

err := c.Validate()
fmt.Println(err)
// Output:
// Address: (State: must be in a valid format.); Email: must be a valid email address.
```

Sometimes, you may want to skip the invocation of a type's `Validate` method. To do so, simply associate
a `valid.Skip` rule with the value being validated.

### Maps/Slices/Arrays of Validatables

When validating an iterable (map, slice, or array), whose element type implements the `valid.Validatable` interface,
the `valid.Validate` method will call the `Validate` method of every non-nil element.
The validation errors of the elements will be returned as `valid.Errors` which maps the keys of the
invalid elements to their corresponding validation errors. For example,

```go
addresses := []Address{
	Address{State: "MD", Zip: "12345"},
	Address{Street: "123 Main St", City: "Vienna", State: "VA", Zip: "12345"},
	Address{City: "Unknown", State: "NC", Zip: "123"},
}
err := valid.Validate(addresses)
fmt.Println(err)
// Output:
// 0: (City: cannot be blank; Street: cannot be blank.); 2: (Street: cannot be blank; Zip: must be in a valid format.).
```

When using `valid.ValidateStruct` to validate a struct, the above validation procedure also applies to those struct 
fields which are map/slices/arrays of validatables. 

#### Each

The `Each` validation rule allows you to apply a set of rules to each element of an array, slice, or map.

```go
type Customer struct {
    Name      string
    Emails    []string
}

func (c Customer) Validate() error {
    return valid.ValidateStruct(&c,
        // Name cannot be empty, and the length must be between 5 and 20.
		valid.Field(&c.Name, valid.Required, valid.Length(5, 20)),
		// Emails are optional, but if given must be valid.
		valid.Field(&c.Emails, valid.Each(is.Email)),
    )
}

c := Customer{
    Name:   "Qiang Xue",
    Emails: []Email{
        "valid@example.com",
        "invalid",
    },
}

err := c.Validate()
fmt.Println(err)
// Output:
// Emails: (1: must be a valid email address.).
```

### Pointers

When a value being validated is a pointer, most validation rules will validate the actual value pointed to by the pointer.
If the pointer is nil, these rules will skip the valid.

An exception is the `valid.Required` and `valid.NotNil` rules. When a pointer is nil, they
will report a validation error.


### Types Implementing `sql.Valuer`

If a data type implements the `sql.Valuer` interface (e.g. `sql.NullString`), the built-in validation rules will handle
it properly. In particular, when a rule is validating such data, it will call the `Value()` method and validate
the returned value instead.


### Required vs. Not Nil

When validating input values, there are two different scenarios about checking if input values are provided or not.

In the first scenario, an input value is considered missing if it is not entered or it is entered as a zero value
(e.g. an empty string, a zero integer). You can use the `valid.Required` rule in this case.

In the second scenario, an input value is considered missing only if it is not entered. A pointer field is usually
used in this case so that you can detect if a value is entered or not by checking if the pointer is nil or not.
You can use the `valid.NotNil` rule to ensure a value is entered (even if it is a zero value).


### Embedded Structs

The `valid.ValidateStruct` method will properly validate a struct that contains embedded structs. In particular,
the fields of an embedded struct are treated as if they belong directly to the containing struct. For example,

```go
type Employee struct {
	Name string
}

type Manager struct {
	Employee
	Level int
}

m := Manager{}
err := valid.ValidateStruct(&m,
	valid.Field(&m.Name, valid.Required),
	valid.Field(&m.Level, valid.Required),
)
fmt.Println(err)
// Output:
// Level: cannot be blank; Name: cannot be blank.
```

In the above code, we use `&m.Name` to specify the validation of the `Name` field of the embedded struct `Employee`.
And the validation error uses `Name` as the key for the error associated with the `Name` field as if `Name` a field
directly belonging to `Manager`.

If `Employee` implements the `valid.Validatable` interface, we can also use the following code to validate
`Manager`, which generates the same validation result:

```go
func (e Employee) Validate() error {
	return valid.ValidateStruct(&e,
		valid.Field(&e.Name, valid.Required),
	)
}

err := valid.ValidateStruct(&m,
	valid.Field(&m.Employee),
	valid.Field(&m.Level, valid.Required),
)
fmt.Println(err)
// Output:
// Level: cannot be blank; Name: cannot be blank.
```


### Conditional Validation

Sometimes, we may want to validate a value only when certain condition is met. For example, we want to ensure the 
`unit` struct field is not empty only when the `quantity` field is not empty; or we may want to ensure either `email`
or `phone` is provided. The so-called conditional validation can be achieved with the help of `valid.When`.
The following code implements the aforementioned examples:

```go
result := valid.ValidateStruct(&a,
    valid.Field(&a.Unit, valid.When(a.Quantity != "", valid.Required).Else(valid.Nil)),
    valid.Field(&a.Phone, valid.When(a.Email == "", valid.Required.Error('Either phone or Email is required.')),
    valid.Field(&a.Email, valid.When(a.Phone == "", valid.Required.Error('Either phone or Email is required.')),
)
```

Note that `valid.When` and `valid.When.Else` can take a list of validation rules. These rules will be executed only when the condition is true (When) or false (Else).

The above code can also be simplified using the shortcut `valid.Required.When`:

```go
result := valid.ValidateStruct(&a,
    valid.Field(&a.Unit, valid.Required.When(a.Quantity != ""), valid.Nil.When(a.Quantity == "")),
    valid.Field(&a.Phone, valid.Required.When(a.Email == "").Error('Either phone or Email is required.')),
    valid.Field(&a.Email, valid.Required.When(a.Phone == "").Error('Either phone or Email is required.')),
)
```

### Customizing Error Messages

All built-in validation rules allow you to customize their error messages. To do so, simply call the `Error()` method
of the rules. For example,

```go
data := "2123"
err := valid.Validate(data,
	valid.Required.Error("is required"),
	valid.Match(regexp.MustCompile("^[0-9]{5}$")).Error("must be a string with five digits"),
)
fmt.Println(err)
// Output:
// must be a string with five digits
```

You can also customize the pre-defined error(s) of a built-in rule such that the customization applies to *every*
instance of the rule. For example, the `Required` rule uses the pre-defined error `ErrRequired`. You can customize it
during the application initialization:
```go
valid.ErrRequired = valid.ErrRequired.SetMessage("the value is required") 
```

### Error Code and Message Translation

The errors returned by the validation rules implement the `Error` interface which contains the `Code()` method 
to provide the error code information. While the message of a validation error is often customized, the code is immutable.
You can use error code to programmatically check a validation error or look for the translation of the corresponding message.

If you are developing your own validation rules, you can use `valid.NewError()` to create a validation error which
implements the aforementioned `Error` interface.

## Creating Custom Rules

Creating a custom rule is as simple as implementing the `valid.Rule` interface. The interface contains a single
method as shown below, which should validate the value and return the validation error, if any:

```go
// Validate validates a value and returns an error if validation fails.
Validate(value interface{}) error
```

If you already have a function with the same signature as shown above, you can call `valid.By()` to turn
it into a validation rule. For example,

```go
func checkAbc(value interface{}) error {
	s, _ := value.(string)
	if s != "abc" {
		return errors.New("must be abc")
	}
	return nil
}

err := valid.Validate("xyz", valid.By(checkAbc))
fmt.Println(err)
// Output: must be abc
```

If your validation function takes additional parameters, you can use the following closure trick:

```go
func stringEquals(str string) valid.RuleFunc {
	return func(value interface{}) error {
		s, _ := value.(string)
        if s != str {
            return errors.New("unexpected string")
        }
        return nil
    }
}

err := valid.Validate("xyz", valid.By(stringEquals("abc")))
fmt.Println(err)
// Output: unexpected string
```


### Rule Groups

When a combination of several rules are used in multiple places, you may use the following trick to create a 
rule group so that your code is more maintainable.

```go
var NameRule = []valid.Rule{
	valid.Required,
	valid.Length(5, 20),
}

type User struct {
	FirstName string
	LastName  string
}

func (u User) Validate() error {
	return valid.ValidateStruct(&u,
		valid.Field(&u.FirstName, NameRule...),
		valid.Field(&u.LastName, NameRule...),
	)
}
```

In the above example, we create a rule group `NameRule` which consists of two validation rules. We then use this rule
group to validate both `FirstName` and `LastName`.


## Context-aware Validation

While most validation rules are self-contained, some rules may depend dynamically on a context. A rule may implement the
`valid.RuleWithContext` interface to support the so-called context-aware valid.
 
To validate an arbitrary value with a context, call `valid.ValidateWithContext()`. The `context.Conext` parameter 
will be passed along to those rules that implement `valid.RuleWithContext`.

To validate the fields of a struct with a context, call `valid.ValidateStructWithContext()`. 

You can define a context-aware rule from scratch by implementing both `valid.Rule` and `valid.RuleWithContext`. 
You can also use `valid.WithContext()` to turn a function into a context-aware rule. For example,


```go
rule := valid.WithContext(func(ctx context.Context, value interface{}) error {
	if ctx.Value("secret") == value.(string) {
	    return nil
	}
	return errors.New("value incorrect")
})
value := "xyz"
ctx := context.WithValue(context.Background(), "secret", "example")
err := valid.ValidateWithContext(ctx, value, rule)
fmt.Println(err)
// Output: value incorrect
```

When performing context-aware validation, if a rule does not implement `valid.RuleWithContext`, its
`valid.Rule` will be used instead.


## Built-in Validation Rules

The following rules are provided in the `validation` package:

* `In(...interface{})`: checks if a value can be found in the given list of values.
* `NotIn(...interface{})`: checks if a value is NOT among the given list of values.
* `Length(min, max int)`: checks if the length of a value is within the specified range.
  This rule should only be used for validating strings, slices, maps, and arrays.
* `RuneLength(min, max int)`: checks if the length of a string is within the specified range.
  This rule is similar as `Length` except that when the value being validated is a string, it checks
  its rune length instead of byte length.
* `Min(min interface{})` and `Max(max interface{})`: checks if a value is within the specified range.
  These two rules should only be used for validating int, uint, float and time.Time types.
* `Match(*regexp.Regexp)`: checks if a value matches the specified regular expression.
  This rule should only be used for strings and byte slices.
* `Date(layout string)`: checks if a string value is a date whose format is specified by the layout.
  By calling `Min()` and/or `Max()`, you can check additionally if the date is within the specified range.
* `Required`: checks if a value is not empty (neither nil nor zero).
* `NotNil`: checks if a pointer value is not nil. Non-pointer values are considered valid.
* `NilOrNotEmpty`: checks if a value is a nil pointer or a non-empty value. This differs from `Required` in that it treats a nil pointer as valid.
* `Nil`: checks if a value is a nil pointer.
* `Empty`: checks if a value is empty. nil pointers are considered valid.
* `Skip`: this is a special rule used to indicate that all rules following it should be skipped (including the nested ones).
* `MultipleOf`: checks if the value is a multiple of the specified range.
* `Each(rules ...Rule)`: checks the elements within an iterable (map/slice/array) with other rules.
* `When(condition, rules ...Rule)`: validates with the specified rules only when the condition is true.
* `Else(rules ...Rule)`: must be used with `When(condition, rules ...Rule)`, validates with the specified rules only when the condition is false.

The `is` sub-package provides a list of commonly used string validation rules that can be used to check if the format
of a value satisfies certain requirements. Note that these rules only handle strings and byte slices and if a string
 or byte slice is empty, it is considered valid. You may use a `Required` rule to ensure a value is not empty.
Below is the whole list of the rules provided by the `is` package:

* `Email`: validates if a string is an email or not. It also checks if the MX record exists for the email domain.
* `EmailFormat`: validates if a string is an email or not. It does NOT check the existence of the MX record.
* `URL`: validates if a string is a valid URL
* `RequestURL`: validates if a string is a valid request URL
* `RequestURI`: validates if a string is a valid request URI
* `Alpha`: validates if a string contains English letters only (a-zA-Z)
* `Digit`: validates if a string contains digits only (0-9)
* `Alphanumeric`: validates if a string contains English letters and digits only (a-zA-Z0-9)
* `UTFLetter`: validates if a string contains unicode letters only
* `UTFDigit`: validates if a string contains unicode decimal digits only
* `UTFLetterNumeric`: validates if a string contains unicode letters and numbers only
* `UTFNumeric`: validates if a string contains unicode number characters (category N) only
* `LowerCase`: validates if a string contains lower case unicode letters only
* `UpperCase`: validates if a string contains upper case unicode letters only
* `Hexadecimal`: validates if a string is a valid hexadecimal number
* `HexColor`: validates if a string is a valid hexadecimal color code
* `RGBColor`: validates if a string is a valid RGB color in the form of rgb(R, G, B)
* `Int`: validates if a string is a valid integer number
* `Float`: validates if a string is a floating point number
* `UUIDv3`: validates if a string is a valid version 3 UUID
* `UUIDv4`: validates if a string is a valid version 4 UUID
* `UUIDv5`: validates if a string is a valid version 5 UUID
* `UUID`: validates if a string is a valid UUID
* `CreditCard`: validates if a string is a valid credit card number
* `ISBN10`: validates if a string is an ISBN version 10
* `ISBN13`: validates if a string is an ISBN version 13
* `ISBN`: validates if a string is an ISBN (either version 10 or 13)
* `JSON`: validates if a string is in valid JSON format
* `ASCII`: validates if a string contains ASCII characters only
* `PrintableASCII`: validates if a string contains printable ASCII characters only
* `Multibyte`: validates if a string contains multibyte characters
* `FullWidth`: validates if a string contains full-width characters
* `HalfWidth`: validates if a string contains half-width characters
* `VariableWidth`: validates if a string contains both full-width and half-width characters
* `Base64`: validates if a string is encoded in Base64
* `DataURI`: validates if a string is a valid base64-encoded data URI
* `E164`: validates if a string is a valid E164 phone number (+19251232233)
* `CountryCode2`: validates if a string is a valid ISO3166 Alpha 2 country code
* `CountryCode3`: validates if a string is a valid ISO3166 Alpha 3 country code
* `DialString`: validates if a string is a valid dial string that can be passed to Dial()
* `MAC`: validates if a string is a MAC address
* `IP`: validates if a string is a valid IP address (either version 4 or 6)
* `IPv4`: validates if a string is a valid version 4 IP address
* `IPv6`: validates if a string is a valid version 6 IP address
* `Subdomain`: validates if a string is valid subdomain
* `Domain`: validates if a string is valid domain
* `DNSName`: validates if a string is valid DNS name
* `Host`: validates if a string is a valid IP (both v4 and v6) or a valid DNS name
* `Port`: validates if a string is a valid port number
* `MongoID`: validates if a string is a valid Mongo ID
* `Latitude`: validates if a string is a valid latitude
* `Longitude`: validates if a string is a valid longitude
* `SSN`: validates if a string is a social security number (SSN)
* `Semver`: validates if a string is a valid semantic version

## Credits

The `is` sub-package wraps the excellent validators provided by the [govalidator](https://github.com/asaskevich/govalidator) package.
