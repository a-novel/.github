# Code quality

## Linting

All code files are linted and formatted. For this purpose, each service provides the following Makefile targets:

```shell
# Just run static checks
make lint
# Try to fix as much linting issues as possible.
make format
```

> The openapi spec has a separate target for linting and formatting, please use `make openapi-lint`

The CI validates a project's code compliance to the rules defined by the linters. Any non-compliance will result in a
failed build.

It is also important that linters cover as much file types as possible. For this, extra linters must be defined for
files that are not covered by the default Go linter.

- [Prettier](https://prettier.io/): initially a JS tool, used to format / lint most non-Go files (.md, .yaml, etc.)
- [SQLFluff](https://sqlfluff.com/): to lint and format SQL files

Those tools should be run by the make targets, so you have nothing to worry about.

## Testing

We put a strong emphasis on rigorously testing every new business logic. Testing prevents accidental regressions, and
also helps immensely in evaluating the true impact of a change.

Go provides a standard testing framework. Tests are written using the
[table-driven style](https://go.dev/wiki/TableDrivenTests), while some exceptions may be allowed depending on the
context.

To efficiently write a table driven test, we recommend (and use) the following pattern:

#### 1 - Define a name, inputs, and expected outputs for the method you are testing.

Name the inputs and outputs accordingly, so they are easily recognizable. We recomment using the argument names for
inputs, and the return value names with the `expect` prefix for outputs.

```go
type testCase struct {
	name	 string
	
	input1	Input1Type
	input2	Input2Type
	
	expect1 Output1Type
	expect2 Output2Type
}
```

#### 2 - Write test loop

Once you have test data ready, it is time to write the test loop.

```go
t.Parallel()

testCases := struct{
	// Your test case definition here.
}{
	{
		// test case 1
	}
	//...
}

for _, testCase := range testCases {
	t.Run(testCase.name, func (t *testing.T) {
		t.Parallel()
		
		output1, output2, err := YourMethod(
			testCase.input1,
			testCase.input2,
		)
		
		require.Equal(t, testCase.expectOutput1, output1)
		require.Equal(t, testCase.expectOutput2, output2)
		require.ErrorIs(t, err, testCase.expectError)
	})
}
```

A test loop just runs each test and validates their outputs.