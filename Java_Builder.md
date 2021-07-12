```java
public class Test {

    private int a;
    private String b;
    
    // getters and setters

    private Test(int a, String b) {
        this.a = a;
        this.b = b;
    }

    public static TestBuilder builder() {
        return new TestBuilder();
    }

    public static class TestBuilder {
        private int a;
        private String b;

        public TestBuilder() {
        }

        public TestBuilder a(int a) {
            this.a = a;
            return this;
        }

        public TestBuilder b(String b) {
            this.b = b;
            return this;
        }

        public Test build() {
            return new Test(a, b);
        }
    }
}
```

