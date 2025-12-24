Mocking can be considered as isolating test by removing the dependencies and replacing them with "test doubles". Mocking usually done by extending a class and mocking some its method - overriding them with methods which either do nothing or implement simulated behavior. 
Mocks can track themselves which can be used to check was a required method called, how many times, what parameters was passed etc. 

Doubles can be 
+ _Fakes_ - working implementations but they use some shortcuts, thus they not suitable for productions 
+ _Stubs_ - respond with pre-defined data.
+ _Mocks_ - also can respond with pre-defined data, but they can throw exceptions, call other methods.

## Methods mocking
For mocking methods instead manually overriding method can be used special macro. There actually modern version and legacy version: 
+ Modern: `{cpp}MOCK_METHOD(ReturnType, MethodName, (Arguments..), (Specs..)`, where specs are `const`, `override`, `noexcept` (keywords between parameters list and body). 
```cpp
bool isReady() const;
MOCK_METHOD(bool, isReady, (), (const));
```
+ Legacy: 
	+ `{cpp}MOCK_METHODn(name, returnType(paramType1, paramType2...)` where n - number of arguments.
	+ `{cpp}MOCK_CONST_METHODn` - for const functions 
```cpp
bool isReady();
MOCK_METHOD0(isReady, bool());

int sum (int a, int b) const;
MOCK_CONST_METHOD2(sum, int(int, int));
```

If a return type or arguments is complex have comma (like `std::pair` or `std::map`) - we but such type inside brackets to avoid compilation error. 

`{cpp}EXPECT_CALL()` function allows to check was a function from mocked class called or not. This assertion set before potential place of call. Also, if call assertion not set, GMock will provide in test output information as warning `Uninteresting mock function call - returning directly`.

```cpp
// Interface for different DB adapters
class IDbConnection {
public:
    IDbConnection(std::string serverAddr) : m_serverAddr(serverAddr) {}
    virtual ~IDbConnection() = default;

    virtual float getSalary(int id) const = 0;
    virtual void updateSalary(int id, float newSalary) const = 0;

    virtual void connect() { std::print("Connecting to *real* DB"); }
    virtual void disconnect() { std::print("Disconnecting to *real* DB"); }

protected:
    std::string m_serverAddr{};
};

class EmployeeManager {
public:
    explicit EmployeeManager(IDbConnection *dbCon) : m_dbCon(dbCon) {  m_dbCon->connect();  }
    ~EmployeeManager() { m_dbCon->disconnect(); }

    void setSalary(int employeeId, float newSalary) { m_dbCon->updateSalary(employeeId, newSalary); }
    float getSalary(int employeeId) const { return m_dbCon->getSalary(employeeId); }

private:
    IDbConnection *m_dbCon{};
};

// Test.cpp
class TestManager : public ::testing::Test {
    void SetUp() override {
        EXPECT_CALL(m_dbCon, connect());
        EXPECT_CALL(m_dbCon, disconnect());
    }

protected:
    MockDbConnection m_dbCon{"dummyConnection"};
};

TEST_F(TestManager, TestConnection) {
    EmployeeManager manager(&m_dbCon);
}
```
As we can see in this example, `IDbConnection` is abstract class which require implementation, even if assume that we have implementation - this class in `{cpp}connect()`  function perform connection to real DB, which can affect on results of tests, making test not isolated. In this example also mixture of old style methods mock and new style. 

For mocked methods we can set behavior with `ON_CALL`, but it's used not often since `EXPECT_CALL` is combination of `ON_CALL` + expectations. 

We can also specify expectation with clauses:
>[!code-ref]+ `{cpp}.With(multi_argument_matcher)`
> Check arguments with matcher (for example, `::testing::Lt()` will check is the first argument being less than the second) 
> [Matchers Reference from Google Test Docs](https://google.github.io/googletest/reference/matchers.html)

>[!code-ref]+ `{cpp}.Times(cardinality)`           
> How many times function call is expected.
>```sheet
>|Cardinality|Meaning|
>|---|---|
>|`AnyNumber()`|The function can be called any number of times.|
>|`AtLeast(n)`|The function call is expected at least _n_ times.|
>|`AtMost(n)`|The function call is expected at most _n_ times.|
>|`Between(m, n)`|The function call is expected between _m_ and _n_ times, inclusive.|
>|`Exactly(n)` or `n`|The function call is expected exactly _n_ times. If _n_ is 0, the call should never happen.|
>```

>[!code-ref]+ `{cpp}.InSequence(sequences...)` 
>Specifies that the mock function call is expected in a certain sequence.
>The parameter _`sequences...`_ is any number of `Sequence` objects.

>[!code-ref]+ `{cpp}.After(expectations...)`
>Specifies that the mock function call is expected to occur after one or more other calls.
>```cpp
>using ::testing::Expectation;
>...
>Expectation init_x = EXPECT_CALL(my_mock, InitX());
>Expectation init_y = EXPECT_CALL(my_mock, InitY());
>EXPECT_CALL(my_mock, Describe())
>            .After(init_x, init_y);
>```
 
>[!code-ref]+ `{cpp}.WillOnce(action)`
>Specifies the mock function’s actual behavior when invoked, for a single matching function call.  The parameter _`action`_ represents the `Action` object that the function call will perform. ([Actions Reference from Google Test Docs](https://google.github.io/googletest/reference/actions.html))
> Quite commonly used would be `testing::Return(val)`, `testing::Invoke(f)`

>[!code-ref]+ `{cpp}.WillRepeatedly(action)`
>Specifies the mock function’s actual behavior when invoked, for all subsequent matching function calls. Takes effect after the actions specified in the `WillOnce` clauses, if any, have been performed.

>[!code-ref]+ `{cpp}.RetiresOnSaturation();`
>Indicates that the expectation will no longer be active after the expected number of matching function calls has been reached.

With `ON_CALL` possible to use `.With()` and only `.WillByDefault(action)`.

Clauses can be chained, also argument values (if the unknown or irrelevant) can be replaced with wildcard-matcher (`::testing::_`)

```cpp
TEST_F(TestManager, TestSetSalary) {
    EmployeeManager manager(&m_dbCon);
    constexpr int employeeId = 10;
    constexpr float salary = 2000.;

    EXPECT_CALL(m_dbCon, updateSalary(employeeId, salary))
        .Times(::testing::Exactly(1));
    manager.setSalary(employeeId, salary);
}

TEST_F(TestManager, TestGetSalary) {
    EmployeeManager manager(&m_dbCon);
    constexpr int employeeId = 10;
    constexpr float salary = 2000.;
    EXPECT_CALL(m_dbCon, getSalary(::testing::_))
        .Times(::testing::Exactly(1))
        .WillOnce(::testing::Return(salary));
    const auto ret = manager.getSalary(employeeId);

    ASSERT_EQ(ret, salary);
}
```

If mocked method require more complex action, we can define such action with `ACTION(name){}` macro.
```cpp
ACTION(ConnErr) {
    std::print("Maybe some additional action here");
    throw std::runtime_error("dummyError");
}

TEST(ManagerTestSuite, TestConnectionError) {
    MockDbConnection dbCon{"dummyConnection"};
    EXPECT_CALL(dbCon, connect()).WillOnce(ConnErr());
    ASSERT_THROW(EmployeeManager manager(&dbCon), std::runtime_error);
}
```

If instead we want to use some method from mocked class, we can use `std::bind` or lambda expressions with `::testing::InvokeWithoutArgs`
```cpp
TEST(ManagerTestSuite, TestConnectionError) {
    MockDbConnection dbCon{"dummyConnection"};
    // auto boundMethod = std::bind(&MockDbConnection::ConnErr, &dbCon);
    auto boundMethod = [ObjectPtr = &dbCon] { ObjectPtr->ConnErr(); };
    EXPECT_CALL(dbCon, connect()).WillOnce(::testing::InvokeWithoutArgs(boundMethod));
    ASSERT_THROW(EmployeeManager manager(&dbCon), std::runtime_error);
}
```

Matchers can be used with arguments as well, for example
`{cpp}EXCEPT_CALL(someObj, someFunc(::testing::Gt(5)))` will check is a passed argument greater that 5. For string can be used `{cpp}HasSubstr("substr")` and so on. 
Matchers can be combined with `{cpp}AllOf(Gt(5), Le(100), Not(7)` (Argument is $5 \le (arg \ne 7) \le 100$)), `{cpp}AnyOf()`, `{cpp}AllOfArray()`, `{cpp}AnyOfArray()`.
Matchers can be used in assertions as well, like `{cpp}ASSERT_THAT`.

For testing elements of container (for example, `std::vector`) can be used matchers `Each()` and `ElementsAre()`:
```cpp
std::vector<int> generateVec(const int n, const int limit) {
    if (limit <= 0)
        throw std::invalid_argument("limit must be greater than 0");

    std::vector<int> vec(n);
    for (int i = 0; i < n; ++i)
        vec[i]= i % limit;
    return vec;
}

TEST(VecTest, ElementsExact) {
    std::vector<int> vec = generateVec(5, 3);
    ASSERT_THAT(vec, testing::ElementsAre(0, 1, 2, 0, 1));
}

TEST(VecTest, ElementsInRange) {
    std::vector<int> vec = generateVec(5, 3);
    using namespace ::testing;
    ASSERT_THAT(vec, Each(AllOf(Ge(0), Lt(3))));
}
```

 For mocking callbacks we can use `::testing::MockFunction`
 ```cpp
 class IDbConnection {
public:
    IDbConnection(std::string serverAddr) : m_serverAddr(serverAddr) {}
    virtual ~IDbConnection() = default;

    virtual void connect() { if (m_onConnect)  m_onConnect(); }
    virtual void disconnect() { std::print("Disconnecting to *real* DB"); }

    using CallBack = std::function<void()>;
    void setOnConnect(const CallBack& onConnect) { m_onConnect = onConnect; }
protected:
    std::string m_serverAddr{};
    CallBack m_onConnect{};
};

class EmployeeManager {
public:
    explicit EmployeeManager(IDbConnection *dbCon) : m_dbCon(dbCon) { m_dbCon->connect(); }
    ~EmployeeManager() { m_dbCon->disconnect(); }
private:
    IDbConnection *m_dbCon{};
};

class MockDbConnection : public IDbConnection {
public:
    MockDbConnection(std::string serverAddr) : IDbConnection(serverAddr) {}

    MOCK_METHOD(void, disconnect, ());
};


TEST(TestManager, TestConnection) {
    MockDbConnection m_dbCon{"dummyConnection"};
    EXPECT_CALL(m_dbCon, disconnect());
    
    ::testing::MockFunction<void()> mockCallback;
    m_dbCon.setOnConnect(mockCallback.AsStdFunction());
    EXPECT_CALL(mockCallback, Call());

    EmployeeManager manager(&m_dbCon);
}
 ```

For mocking private members in cases  when we want to use them in expectations, we can use trick, making member _private virtual_ in mock class _public_. For static methods it's more complicated and even documentation states that there is no direct way of doing it, but documentation also suggest as work around use another class and friendship
```cpp
class TestHelper {
    virtual void increaseConnectionCount();
    friend class IDbConnection;
};

class IDbConnection {
public:
    IDbConnection(std::string serverAddr, TestHelper* helper) : m_serverAddr(serverAddr), m_helper(helper) {}
    virtual ~IDbConnection() = default;

    virtual void connect();

    virtual void disconnect() { }

protected:
    std::string m_serverAddr{};
private:
    static void increaseConnectionCount() {
        s_connectionCount++;
    }
    static unsigned int s_connectionCount;

    TestHelper* m_helper{};
    friend class TestHelper;
};

unsigned int IDbConnection::s_connectionCount = 0;

// Call private static function of class unders test
void TestHelper::increaseConnectionCount() { IDbConnection::increaseConnectionCount(); }
// instead direct static member call - use "contractor"
void IDbConnection::connect() { m_helper->increaseConnectionCount(); }

class EmployeeManager {
public:
    explicit EmployeeManager(IDbConnection *dbCon) : m_dbCon(dbCon) { m_dbCon->connect(); }
    ~EmployeeManager() { m_dbCon->disconnect(); }
private:
    IDbConnection *m_dbCon{};
};

class MockDbConnection : public IDbConnection {
public:
    MockDbConnection(std::string serverAddr, TestHelper* th) : IDbConnection(serverAddr, th) {}

    MOCK_METHOD(void, disconnect, ());
};

// Mock test helpers as well
class MocTestHelper : public TestHelper {
public:
    MOCK_METHOD(void, increaseConnectionCount, ());
};

TEST(TestManager, TestConnection) {
    MocTestHelper helper;
    EXPECT_CALL(helper, increaseConnectionCount()).Times(1);
    MockDbConnection m_dbCon{"dummyConnection", &helper};
    EXPECT_CALL(m_dbCon, disconnect());

    EmployeeManager manager(&m_dbCon);
}
```