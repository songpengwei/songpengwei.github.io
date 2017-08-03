---
title: 一些设计模式
updated: 2017-08-03 11:01
---
写程序的时候，规模小，尚不能感觉设计模式的重要性。等规模一上来，需求一迭代，一个应用了恰当设计模式的工程，总能以最小的代价进行最快的迭代。
但是一个奇怪的点是，我总记不住具体的实现所对应的设计模式的名字，但是对他们背后的设计思想，却是念念不忘——依赖于抽象而非具体；对扩展开放，对修改关闭；


### Builder

首先，将一个复杂逻辑抽象成一组构建过程（具有前后先后次序，即时序约束）或者一组元操作（便于进行组合实现复杂逻辑），用一个接口封装。
然后，不同的逻辑实体类，继承该接口，进行不同的具体实现。
最后，依赖于接口，组合构建过程或元操作，进行具体业务代码实现。以后想换一个实现，只需要某处换一个具体实现类就行了。

[小例子一枚](https://sourcemaking.com/design_patterns/builder/java/2)：

```Java
/* "Product" */
class Pizza {
    private String dough = "";
    private String sauce = "";
    private String topping = "";

    public void setDough(String dough) {
        this.dough = dough;
    }

    public void setSauce(String sauce) {
        this.sauce = sauce;
    }

    public void setTopping(String topping) {
        this.topping = topping;
    }
}

/* "Abstract Builder" */
abstract class PizzaBuilder {
    protected Pizza pizza;

    public Pizza getPizza() {
        return pizza;
    }

    public void createNewPizzaProduct() {
        pizza = new Pizza();
    }

    public abstract void buildDough();
    public abstract void buildSauce();
    public abstract void buildTopping();
}

/* "ConcreteBuilder" */
class HawaiianPizzaBuilder extends PizzaBuilder {
    public void buildDough() {
        pizza.setDough("cross");
    }

    public void buildSauce() {
        pizza.setSauce("mild");
    }

    public void buildTopping() {
        pizza.setTopping("ham+pineapple");
    }
}

/* "ConcreteBuilder" */
class SpicyPizzaBuilder extends PizzaBuilder {
    public void buildDough() {
        pizza.setDough("pan baked");
    }

    public void buildSauce() {
        pizza.setSauce("hot");
    }

    public void buildTopping() {
        pizza.setTopping("pepperoni+salami");
    }
}

/* "Director" */
class Waiter {
    private PizzaBuilder pizzaBuilder;

    public void setPizzaBuilder(PizzaBuilder pb) {
        pizzaBuilder = pb;
    }

    public Pizza getPizza() {
        return pizzaBuilder.getPizza();
    }

    public void constructPizza() {
        pizzaBuilder.createNewPizzaProduct();
        pizzaBuilder.buildDough();
        pizzaBuilder.buildSauce();
        pizzaBuilder.buildTopping();
    }
}

/* A customer ordering a pizza. */
public class PizzaBuilderDemo {
    public static void main(String[] args) {
        Waiter waiter = new Waiter();
        PizzaBuilder hawaiianPizzabuilder = new HawaiianPizzaBuilder();
        PizzaBuilder spicyPizzaBuilder = new SpicyPizzaBuilder();

        waiter.setPizzaBuilder( hawaiianPizzabuilder );
        waiter.constructPizza();

        Pizza pizza = waiter.getPizza();
    }
}

```

这就是一个典型的依赖于抽象而非具体。





