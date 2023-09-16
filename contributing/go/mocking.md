# Mocking

We use [Mockery](https://github.com/vektra/mockery) to generate mocks for our dependencies. You can update all mocks.
when working on new features, by running the following make target:

```shell
make mocks
```

## Work with mocks

When writing a [test loop](code-quality.md#2---write-test-loop), you have to also define what dependencies are expected to be called,
and what they should return.

In your test data, you should first define what each mocked dependency should return:

```go
type MockDependency1Data struct {
	output1 DependencyOutput1Type
	output2 DependencyOutput2Type
}
```

Then, you can use this data in your test case definition:

```go
type testCase struct {
	name	 string
	
	input1	Input1Type
	input2	Input2Type
	
	mockDependency1Data *MockDependency1Data
	
	expect1 Output1Type
	expect2 Output2Type
}
```

> It is important to use a pointer, so you can pass nil values to indicate your dependency should not be called.

Finally, in your test loop, you can set up the mock service to return the expected data:

```go
for _, testCase := range testCases {
	t.Run(testCase.name, func (t *testing.T) {
		t.Parallel()
		
		mockDependency1 := mocks.NewService1(t)
		
		if testCase.mockDependency1Data != nil {
			mockDependency1.EXPECT().
				Method1(arg1, arg2).
				Return(testCase.mockDependency1Data.output1, testCase.mockDependency1Data.output2)
		}
		
		output1, output2, err := YourMethod(
			testCase.input1,
			testCase.input2,
		)
		
		require.Equal(t, testCase.expectOutput1, output1)
		require.Equal(t, testCase.expectOutput2, output2)
		require.ErrorIs(t, err, testCase.expectError)

		mockDependency1.AssertExpectations(t)
	})
}
```

A few things to consider:

- The last line `mockDependency1.AssertExpectations(t)` is important for each mock, to ensure it was called as expected.
- The arguments passed to the mock method should be computed during the test. If you need to define a mock
  argument manually, pass it through the data object.

The arguments used to call the mock should be statically known. If this is not possible, you have 2 choices:

- Use the `mock.MatchedBy` method:

```go
for _, testCase := range testCases {
	t.Run(testCase.name, func (t *testing.T) {
		t.Parallel()
		
		mockDependency1 := mocks.NewService1(t)
		
		if testCase.mockDependency1Data != nil {
			mockDependency1.EXPECT().
				Method1(
					arg1, 
					// Here, arg2 has a time-dependent value.
					mock.MatchedBy(func(arg2 Input2Type) bool {
						return arg2.ID == expectArg2ID && 
							arg2.CreatedAt.After(time.Now().Add(-time.Minute))
					},
				).
				Return(testCase.mockDependency1Data.output1)
		}
		
		output1, output2, err := YourMethod(
			testCase.input1,
			testCase.input2,
		)
		
		require.Equal(t, testCase.expectOutput1, output1)
		require.Equal(t, testCase.expectOutput2, output2)
		require.ErrorIs(t, err, testCase.expectError)

		mockDependency1.AssertExpectations(t)
	})
}
```

- Use the static value `mock.Anything`, if a check using `mock.MatchedBy` is not possible
  (for example, a custom context)