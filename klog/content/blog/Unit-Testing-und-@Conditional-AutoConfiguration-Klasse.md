+++
title = "Wie testet man die @Conditional in Spring Boot AutoConfiguration-Klassen?"
date = "2024-01-07T16:27:33+03:00"

#
# description is optional
#
description = "Um die Arbeit mit Spring Boot zu vereinfachen, wird Spring-Boot-Starter verwendet. Durch die Definition der AutoConfiguration-Klassen kann man eigene Starters erstellen. In diesem Tutorial kann man erfahren, wie man Unit-Tests für diese Klassen erstellt."

tags = ["Spring Boot","Unit-Testing","Junit5","Mockito"]
+++

# Umgebung

* Java 21
* Spring Boot 3.2.1
* Junit 5.10.1
* Mockito 5.7.0
* AssertJ 3.24.0

# Einführung

„AutoConfiguration“ in Spring Boot ist ein Feature, das auf bedingter Konfiguration basiert, um verschiedene Komponenten und Funktionen einer Spring-Application automatisch zu konfigurieren. Dies wird durch die bedingte Annotation „@Conditional“ ermöglicht. Mithilfe diesen Bedingten können Konfigurationen auf bestimmten Bedingungen aktiviert oder deaktiviert werden. Eine Frage, wie stellt man sicher, dass die definierten Bedingungen korrekt sind, wenn man ein eigenen Spring-Boot-Starter erstellen möchte? Auf dem Instinkt kann man direkt die Application starten und debuggen. Mit diesem Verhalten kann man natürlich diese Aufgabe erledigen, jedoch hilft es keineswegs bei der Automatisierung. Eine bessere Alternativ dazu kann man Unit-Testing mit Junit und Mockito für AutoConfiguration-Klasse erstellen, in diesem Tutorial kannst du ein paar Beispiele erfahren.

# Grundwissen

## ApplicationContextRunner

ApplicationContextRunner ist ein praktisches Werkzeug beim Testen des Spring-Anwendungskontexts. Es ermöglicht die Erstellung eines minimalen Spring-Kontexts während des Tests, der nur die erforderlichen Beans enthält.

# Annotationen und Tests

## @ConditionalOnClass

### Configuration

```java
@Bean
@ConditionalOnClass(Logger.class)
public Marker conditionalOnClassMarker() {
    return new Marker();
}
```

### Test

```java
@Test
void conditionalOnClassMarker() {
    new ApplicationContextRunner()
        .withConfiguration(AutoConfigurations.of(ClassConditional.class))
        .run(
            (context) -> {
                assertThat(context).hasSingleBean(ClassConditional.Marker.class);
                assertThat(context)
                    .getBean("conditionalOnClassMarker")
                    .isSameAs(context.getBean(ClassConditional.Marker.class));
            });
}
```

## @ConditionalOnMissingClass

### Configuration

```java
@Bean
@ConditionalOnMissingClass("org.apache.logging.log4j.Logger")
public Marker conditionalOnMissingClassMarker() {
    return new Marker();
}
```

### Test

Mit **FilteredClassLoader** kann eine Umgebung ohne die bestimmten Klassen simuliert werden.

```java
@Test
void conditionalOnMissingClassMarker() {
    new ApplicationContextRunner()
        .withConfiguration(AutoConfigurations.of(ClassConditional.class))
        .withClassLoader(new FilteredClassLoader(Logger.class))
        .run(
            (context) -> {
                assertThat(context).hasSingleBean(ClassConditional.Marker.class);
                assertThat(context)
                    .getBean("conditionalOnMissingClassMarker")
                    .isSameAs(context.getBean(ClassConditional.Marker.class));
            });
}
```

## @ConditionalOnBean

### Configuration

```java
@Bean
@ConditionalOnBean(Logger.class)
public Marker conditionalOnBeanMarker() {
    return new Marker();
}
```

### Test

Für diesen Fall wird eine registrierte Bean im Kontext benötigt, deren Typ „Logger“ ist. Aber es gibt keine. Um diese Situation zu simulieren, füge einfach in der Testklasse mithilfe einer Konfigurationsklasse eine Instanz ein, die durch **Mockito** erstellt wird.

```java
@Test
void conditionalOnBeanMarker() {
    new ApplicationContextRunner()
        .withConfiguration(
            AutoConfigurations.of(BeanConditional.class, MockLoggerAutoConfiguration.class))
        .run(
            (context) -> {
                assertThat(context).hasSingleBean(BeanConditional.Marker.class);
                assertThat(context)
                    .getBean("conditionalOnBeanMarker")
                    .isSameAs(context.getBean(BeanConditional.Marker.class));
            });
}

@Configuration
@AutoConfigureBefore(BeanConditional.class)
static class MockLoggerAutoConfiguration {

    @Bean
    public Logger logger() {
        return mock(Logger.class);
    }
}
```

## @ConditionalOnMissingBean

### Configuration

```java
@Bean
@ConditionalOnMissingBean(Marker.class)
public Marker conditionalOnMissingBeanMarker() {
    return new Marker();
}
```

### Test

Eine Mock-Bean ist auch hilfreich, wenn man in der Konfiguration-Klasse **@ConditionalOnMissingBean** verwendet hat. Wir können auch in der Testklasse mithilfe einer Konfigurationklasse eine fiktive Bean einfügen. In diesem Fall ist es nicht zwingend erforderlich, die Wahrheit der Behauptung zu überprüfen, ob die Bean im Kontext eine Mockbean ist, weil die beiden Beans unterschiedliche Namen haben. Aber die Beans werden denselben Namen haben, wenn man mit **BeanName** diese Annotation verwendet. Hierbei wird die statische Methode **isMock()** eine bedeutende Rolle spielen.

```java
@Test
void conditionalOnMissingBeanMarker() {
    new ApplicationContextRunner()
        .withConfiguration(AutoConfigurations.of(BeanConditional.class))
        .run(
            (context) -> {
            assertThat(context).hasSingleBean(BeanConditional.Marker.class);
            assertThat(context)
                .getBean("conditionalOnMissingBeanMarker")
                .isSameAs(context.getBean(BeanConditional.Marker.class));
            assertThat(isMock(context.getBean(BeanConditional.Marker.class))).isFalse();
            });
}

@Test
void conditionalOnMissingBeanWithMockMarker() {
    new ApplicationContextRunner()
        .withConfiguration(
            AutoConfigurations.of(BeanConditional.class, MockMarkerAutoConfiguration.class))
        .run(
            (context) -> {
            assertThat(context).hasSingleBean(BeanConditional.Marker.class);
            assertThat(context)
                .getBean("mockMarker")
                .isSameAs(context.getBean(BeanConditional.Marker.class));
            assertThat(isMock(context.getBean(BeanConditional.Marker.class))).isTrue();
            });
}

@Configuration
@AutoConfigureBefore(BeanConditional.class)
static class MockMarkerAutoConfiguration {

    @Bean
    public BeanConditional.Marker mockMarker() {
        return mock(BeanConditional.Marker.class);
    }
}
```

## @ConditionalOnProperty

### Configuration

```java
@Bean
@ConditionalOnProperty(value = "conditional.enabled", havingValue = "true", matchIfMissing = true)
public Marker conditionalOnPropertyMarker() {
    return new Marker();
}
```

### Test

Die Methode **ApplicationContextRunner.withPropertyValues(String... pairs)** vereinfacht die Simulation der PropertyConditional.

```java
@Test
void conditionalOnPropertyMarker() {
    new ApplicationContextRunner()
        .withConfiguration(AutoConfigurations.of(PropertyConditional.class))
        .run(
            (context) -> {
            assertThat(context).hasSingleBean(PropertyConditional.Marker.class);
            assertThat(context)
                .getBean("conditionalOnPropertyMarker")
                .isSameAs(context.getBean(PropertyConditional.Marker.class));
            });
}

@Test
void noConditionalOnPropertyMarker() {
    new ApplicationContextRunner()
        .withConfiguration(AutoConfigurations.of(PropertyConditional.class))
        .withPropertyValues("conditional.enabled=false")
        .run(
            (context) -> {
            assertThat(context).doesNotHaveBean(PropertyConditional.Marker.class);
            });
}
```

## Kombination von @Conditional

In Spring Boot gibt es Möglichkeiten, verschiedene Bedingungen zu kombinieren. Durch die Kombination der oben genannten Beispiele lassen sich problemlos die meisten Szenarien abdecken. 

# Code auf Github
[ksewen/unit-testing-tutorial/conditional](https://github.com/ksewen/unit-testing-tutorial/tree/main/conditional)

Diskussionen, Fragen um Aufzeigen meiner Fehler sind willkommen.