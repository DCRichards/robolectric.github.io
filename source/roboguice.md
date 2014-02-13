---
layout: default
title: Integrating RoboGuice and Robolectric
---

# Integrating RoboGuice and Robolectric
The [RobolectricSample application](https://github.com/robolectric/RobolectricSample) includes an
example of how to test Android applications that are built using the [RoboGuice](http://code.google.com/p/roboguice/)
dependency injection framework. This article explains what we did and how you can add RoboGuice to your own projects.

### The Sample Activity
For the most part RoboGuice will just work when injected instances of classes are exercised by unit tests, but with a
little bit of work can inject those instances directly into our tests. As an example, we can start with this simple
activity:

```java
public class InjectedActivity extends GuiceActivity {
    @InjectResource(R.string.injected_activity_caption) String caption;
    @InjectView(R.id.injected_text_view) TextView injectedTextView;

    @Override protected void onCreate(Bundle savedInstanceState) {
        super.onCreate(savedInstanceState);
        setContentView(R.layout.injected);
        injectedTextView.setText(caption);
    }
}
```

To get this to work you need to change your manifest to point to <code>roboguice.application.GuiceApplication</code>
(or a subclass) and link with the RoboGuice library. We wrote the following test:

```java
@Test
public void shouldAssignStringToTextView() throws Exception {
    injectedActivity.onCreate(null);
    TextView injectedTextView =
        (TextView) injectedActivity.findViewById(R.id.injected_text_view);
    assertEquals(injectedTextView.getText(), caption);
}
```

This test passes but the set up for it is ugly:

```java
InjectedActivity injectedActivity;
String caption;
@Before
public void setUp() {
    injectedActivity = new InjectedActivity();
    Resources resources = injectedActivity.getResources();
    caption = resources.getString(R.string.injected_activity_caption);
}
```

### Injecting Tests
With some changes to the test runner we can make it look like this:

```java
    @Inject InjectedActivity injectedActivity;
    @InjectResource(R.string.injected_activity_caption) String caption;
```

In <code>RobolectricSample</code> we created a subclass of <code>RobolectricTestRunner</code> called
<code>InjectedTestRunner</code> and extended it to ensure that instances of the test class were
injected before the tests were run. <code>RobolectricTestRunner</code> has a method called
<code>prepareTest(Test test)</code> that exists for the purpose of giving its subclasses access to the
<code>Test</code> object just before each test. It can be used for injection as follows:

```java
public class InjectedTestRunner extends RobolectricTestRunner {
    public InjectedTestRunner(Class<?> testClass) throws InitializationError {
        super(testClass);
    }

    @Override public void prepareTest(Object test) {
        GuiceApplication sampleApplication = (GuiceApplication) Robolectric.application;
        Injector injector = sampleApplication.getInjector();
        injector.injectMembers(test);
    }
}
```

It can be used in a <code>@RunWith</code> clause instead of <code>RobolectricTestRunner</code>:

```java
@RunWith(InjectedTestRunner.class)
public class InjectedActivityTest {
...
}
```

### Injecting <code>Context</code> Objects onto Tests
There are some types, such as <code>Context</code>s that RoboGuice can't inject without entering the context scope
in the test runner's <code>prepareTest()</code> method:

```java
@Override public void prepareTest(Object test) {
    GuiceApplication sampleApplication = (GuiceApplication) Robolectric.application;
    Injector injector = sampleApplication.getInjector();
    ContextScope scope = injector.getInstance(ContextScope.class);
    scope.enter(sampleApplication);
    injector.injectMembers(test);
}
```

Now it is possible to inject a <code>Context</code> object onto the test:

```java
@Inject Context context;
@Test
public void shouldBeAbleToInjectAContext() throws Exception {
    assertNotNull(context);
}
```

#### [Next: Test Modules](roboguice2.html)