> I slightly changed title and merged together sections "Unit Testing Basics" and "Fixtures" 
## Unit tests characteristics 
Unit test are automated functional regression tests aimed to test _one unit_ - basic block of software (usually one class or one functions).
Unit tests can be executed independently (this require also paying attention to global variables, if one test modifying global variable, that changed shouldn't affect other tests). Unit tests don't rely on external input (like user input) - they are isolated entity. 

## Basic structure of unit tests 
Unit tests in general  divided into 3 parts: 
+ *Arrange* - test setup (create object, set inputs, preconditions etc)
+ *Act* - executing method
+ *Assert* - check results
Sometime there also cleanup phase, but it's not a good practices since if test get failed, cleanup part might not be executed. 

Such structure allows to separate what is tested from setup and verification. This also allows to localize issue, since ideally unit test should test only one thing at once, which will provide more clear information what is failed and under what conditions. 

```cpp
int countPositives(const std::vector<int>& inp) {  
    return static_cast<int>(std::ranges::count_if(inp, [](const auto x) { return x > 0; }));  
}

TEST(TestPositiveCount, HappyPath) {  
    // Arrange  
    const std::vector vec {1, -1, 3};  
  
    // Act  
    const auto res = countPositives(vec);  
  
    // Assert  
    ASSERT_EQ(2, res);  
}  
  
TEST(TestPositiveCount, Empty) {  
    const std::vector<int> vec {};  
      
    const auto res = countPositives(vec);  
      
    ASSERT_EQ(0, res);  
}  
  
TEST(TestPositiveCount, AllNegative) {  
    const std::vector vec {-1, -4, -5};  
      
    const auto res = countPositives(vec);  
  
    ASSERT_EQ(0, res);  
}
```

## Assertions 
Assertions can have two possible outcomes: _success_ or failure. In case of failure we have two types of assertions (as far as behavior is concerned): _fatal failure_ and _non-fatal failure_.
Fatal failure means that if the condition is not satisfied, then the test should stop execution. With non-fatal failure it's possible to continue test. 

Fatal failure assertion can be set with `ASSERT_*`, non-fatal with `EXPECT_*`. 

```sheet
| Fatal                          | Non-Fatal                      | What is tests                                          |
| ------------------------------ | ------------------------------ | ------------------------------------------------------ |
| `ASSERT_TRUE(condition)`  | `EXPECT_TRUE(condition)`  | `condition` is `true`                                  |
| `ASSERT_FALSE(condition)` | `EXPECT_FALSE(condition)` | `condition` is not `true`                              |
| `ASSERT_EQ(x, y)`         | `EXPECT_EQ(x, y)`         | $x == y$                                               |
| `ASSERT_NE(x, y)`         | `EXPECT_NE(x, y)`         | $x \ne y$                                              |
| `ASSERT_LT(x, y)`         | `EXPECT_LT(x, y)`         | $x < y$                                                |
| `ASSERT_LE(x, y)`         | `EXPECT_LE(x, y)`         | $x \le y$                                              |
| `ASSERT_GT(x, y)`         | `EXPECT_GT(x, y)`         | $x > y$                                                |
| `ASSERT_GE(x, y)`         | `EXPECT_GE(x, y)`         | $x \ge y$                                              |
| Assrtions on C (`char*`) strings |<|<|
| `ASSERT_STREQ(x, y)`  | `EXPECT_STREQ(x, y)`  | `x` and `y` have the same content                      |
| `ASSERT_STRNE(x, y)`  | `EXPECT_STRNE(x, y)`  | `x` and `y` have different contents                    |
| `ASSERT_STRCASEEQ(x, y)`  | `EXPECT_STRCASEEQ(x, y)`  | `x` and `y` have the same content (case insensitive)   |
| `ASSERT_STRCASENE(x, y)`  | `EXPECT_STRCASENE(x, y)`  | `x` and `y` have different contents (case insensative) |
| Assertions on execptions |<|<|
| `ASSERT_THROW(statement, exceptionType)` |`EXPECT_THROW(statement, exceptionType)` | `statement` throws an exception of the _exact_ given type|
| `ASSERT_ANY_THROW(statement)` |`EXPECT_ANY_THROW(statement)` | `statement` throws an _any_ exception|
| `ASSERT_NO_THROW(statement)` | `EXPECT_NO_THROW(statement)` | `statement` thows no exception| 
```

As shown in table, for C-string used separate assertions. This is due to fact, that C-strings are pointers to memory where string located and with `ASSERT_EQ` we actually will compare addresses instead strings. 
For `std::string` `ASSERT_EQ` works correctly (due to `operator==` overload). 

## Fixtures
Fixtures allows to reuse the code in the setup and tear-down phases. Fixtures are classes derived from `testing::Test` classes. This class contain functions which can be overwritten for required behavior: 
+ `{cpp}virtual void SetUp()` - function which executed before each test 
+ `{cpp}virtual void TearDown()` - function which executed after each test 
+ `{cpp}static void SetUpTestSuite()` - executed before Test suite
+ `{cpp}static void TearDownTestSuite()` - executed after Test suite. 

Test body created with `TEST_F()` instead `TEST()`. First argument is name of created fixture.
Each test have execution flow `Fixture Constructor -> SetUp -> Test body -> TeadDown -> Fixture Destructor`. Reasons to used `SetUp` and `TearDown` instead constructor and destructor is that inside `SetUp` and `TearDown` possible to use assertions and we can be ensure that fixture initialization complete and since polymorphism is used, it's not possible to use virtual functions in constructors and destructors.

```cpp
class BankAccount {
    double m_balance {};
public:
    [[nodiscard]] double getBalance() const { return m_balance; }
    
    void deposit(const double amount) {
        if (amount < 0) throw std::invalid_argument("Negative amount");
        m_balance += amount;
    }
    
    void withdraw(const double amount) {
        if (amount < 0) throw std::invalid_argument("Negative amount");

        const double nextBalance = m_balance - amount;
        if (nextBalance < 0) throw std::runtime_error("Not enough to withdraw");
        m_balance = nextBalance;
    }
};

class BankAccountTest : public ::testing::Test {
    void SetUp() override {
        // Set up balance before each test
        EXPECT_NO_THROW(m_account.deposit(10.0));
    }
protected:
    // Initialize once for all tests
    BankAccount m_account{};
};

TEST_F(BankAccountTest, HappyPathWithdraw) {
    EXPECT_NO_THROW(m_account.withdraw(5.0));
    EXPECT_EQ(5.0, m_account.getBalance());
}

TEST_F(BankAccountTest, NotEnoughWithdraw) {
    EXPECT_THROW(m_account.withdraw(15.0), std::runtime_error);
}

TEST_F(BankAccountTest, NegativeWithdraw) {
    EXPECT_THROW(m_account.withdraw(-10.0), std::invalid_argument);
}
```


## Parameterized tests 
Parameterized tests another way to reduce duplication in test by using templates. It's useful when in tests used same code but different values. 
Parameterized tests inherited from `{cpp}testing::TestWithParam<T>`, for test body used `TEST_P()`. Function `{cpp}INSTANTIATE_TEST_SUITE_P(prefix, TestSuiteName, generator)` will generate tests from parameters. 
Parameterized tests supports also generators: 
+ `{cpp}testing::Range(T begin, T end, T step)` - generate values in a given range 
+ `{cpp}testing::Values(...)` - set individual values 
+ `{cpp}testing::ValuesIn(..)` - use containers through iterators 
+ `{cpp}testing::Bool()` - generate `true` and `false`
+ `{cpp}testing::Combine()` - combine generators mentioned above using Cartesian product 

With parameterized tests exists one downsides - it's not possible to defined with one test suite several different tests (lets say, happy path and out of range) - for different test should be created different test suite. As a solution, we can change `T` to `{cpp}std::tuple<int, bool>` where first parameter is value and second is expected result and set expectation with generator.

```cpp
class SensorValidator {
    int m_low{};
    int m_high{};
public:
    explicit SensorValidator(const int low, const int high) : m_low(low), m_high(high) {}
    bool inRange(const int val) { return m_low <= val && val <= m_high; }
};

class SensorValidatorTest : public ::testing::TestWithParam<std::tuple<int, bool>> {
protected:
    SensorValidator m_validator {5, 10};
};

TEST_P(SensorValidatorTest, TestInRange) {
    auto [value, expected] = GetParam();

    bool isInside = m_validator.inRange(value);
    ASSERT_EQ(isInside, expected);
}

INSTANTIATE_TEST_SUITE_P(HappyPath, 
                         SensorValidatorTest, 
                         testing::Combine(testing::Range(5, 10, 1), 
                                          testing::Values(true)));
INSTANTIATE_TEST_SUITE_P(OutOfRange, 
                         SensorValidatorTest, 
                         testing::Combine(testing::Values(-1, 0, 11), 
                                          testing::Values(false)));
```