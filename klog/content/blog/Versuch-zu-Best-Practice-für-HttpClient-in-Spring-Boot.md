+++
title = "Versuch Zu Best Practice Für HttpClient"
date = "2024-05-22T14:25:47+03:00"

#
# description is optional
#
description = "In der modernen Architekt der Software sind Http-Anfragen unvermeidbar. Sowohl arbeitet man in einer Micorservice-Umgebung z.B. basiert auf Spring Cloud Framework, in der normalerweise über Http-Protokoll kommunizieren, sondern möchte man auch die externen Services anrufen, die Http-Schnittstellen anbieten. In diesem Artikel kann man erfahren, ein paar häufige Fehler bei Verwendungen für Apache-HttpClient im Praxis und wie man sie beheben kann."

tags = ["HttpClient","Http-Anfrage","Spring Boot","Best Practice",]
+++

Niemand wird überraschen sein, die Aufgabe von Http-Anfrage durchzuführen. Apache HttpClient nimmt eine wichtige Stellung in der Java-Entwicklung ein, obwohl Kapselungswerkzeuge möglicherweise am Arbeitsplatz verfügbar sind, z.B. Feign in Spring Cloud oder SDKs von Anbieter der externen Services, weil sie möglicherweise tatsächlich auf Apache HttpClient basieren. Falsche Nutzung kann jedoch zu ernsthaften Problemen wie Ressourcenlecks, Performanceeinbußen und Sicherheitslücken führen. In diesem Artikel untersucht dann ein paar Beispiele der häufigen Falsche und die besseren Verfahren dafür.

# Bei jeder Anfrage eine neue Instanz erzeugen

Das ist ein häufiges Problem, besonders für Anfänger. Von der offiziellen Dokumentation kopiert man ganz einfach den Beispielcode, und dann, kann es natürlich reibungslos laufen. Aber es führt zu erhöhtem Ressourcenverbrauch und schlechterer Performance, insbesondere in Szenarien mit hohen Leistungsanforderungen oder hoher Parallelität. Stattdessen sollte man eine wiederverwendbare Instanz verwenden:

```java
public class HttpClientHolder {

    private static final CloseableHttpClient httpClient;

    static {
        httpClient = HttpClients.custom()
            .build();
    }

    public static CloseableHttpClient getInstance() {
        return httpClient;
    }

}
```

oder in den Kontext registrieren, wenn man sich mit Spring Framework beschäftigt:

```java
@Configuration(proxyBeanMethods = false)
public class HttpClientAutoConfiguration {

    private CloseableHttpClient httpClient;

    @Bean
    public CloseableHttpClient httpClient() {
        this.httpClient = HttpClients.custom()
            .build();

        return this.httpClient;
    }

    @PreDestroy
    public void destroy() {
        if (httpClient != null) {
            httpClient.close(CloseMode.GRACEFUL);
        }
    }
}
```

# Ohne Konfiguration zur angemessenen Zeitüberschreitung

Das ist eines der häufigsten Probleme in Produktion und kann mitunter zu fatalen Problemen führen. Um z.B. auf die Etablierung der Verbindung zu warten, werden die Faden im System erschöpft sein und wird das System abstürzen. Die Standardwert der Konfiguration zur „connectTimeout“ in diesem Werkzeug als 3 Minuten eingesetzt wird. Aber bei modernen Microservices werden Anfragen meistens innerhalb weniger hundert Millisekunden beantwortet. Selbst eine Anrufe zur externen Service dauert in der Regel nicht mehr als ein paar Sekunden. Es ist sinnvoll, durch [RequestConfig](https://github.com/apache/httpcomponents-client/blob/master/httpclient5/src/main/java/org/apache/hc/client5/http/config/RequestConfig.java) die angemessenen Zeitüberschreitung zu konfigurieren:

```java

RequestConfig requestConfig = RequestConfig.custom()
    .setConnectTimeout(5000)
    .setSocketTimeout(5000)
    .setConnectionRequestTimeout(5000)
    .build();

CloseableHttpClient httpClient = HttpClients.custom()
    .setDefaultRequestConfig(requestConfig)
    .build();

```

# Ohne Verbindungs-Pools

Obwohl es in der Regel mehrere Ressourcen und Zeit verbraucht wird, führt dieses Verhalten selten zu faltalen Problemen, außer dass die Anzahl der Verbindungen erschöpft ist. Erfahrene Entwickler verwenden ständig [PoolingHttpClientConnectionManager](https://github.com/apache/httpcomponents-client/blob/master/httpclient5/src/main/java/org/apache/hc/client5/http/impl/io/PoolingHttpClientConnectionManager.java):

```java

PoolingHttpClientConnectionManager connectionManager = new PoolingHttpClientConnectionManager();
connectionManager.setMaxTotal(100);
connectionManager.setDefaultMaxPerRoute(20);

CloseableHttpClient httpClient = HttpClients.custom()
    .setConnectionManager(connectionManager)
    .build();

```

Eine Implementirung von [ConnectionKeepAliveStrategy](https://github.com/apache/httpcomponents-client/blob/master/httpclient5/src/main/java/org/apache/hc/client5/http/ConnectionKeepAliveStrategy.java) ist manchmal notwendig.

```java

HttpClients.custom().setKeepAliveStrategy(keepAliveStrategy);

```

# Schnittstelle, die nicht idempotent sind, auch erneut versuchen

Netzwerkanfragen können aufgrund vorübergehender Netzwerkprobleme fehlschlagen. Ein Wiederholungsmechanismus erhöht die Zuverlässigkeit der Anfragen. HttpClient bietet eine anpassbare Wiederholungsbehandlung und einfach zu benutzen:

```java

HttpRequestRetryHandler retryHandler = new DefaultHttpRequestRetryHandler(3, true);

```

Aber, in der Tat sind viele Schnittstellen, insbesondere POST-Anfragen, nicht idempotent. Dies kann dazu führen, dass bei einer erneuten Anfrage unvorhersehbare Seiteneffekte auftreten, wie z. B. das Hinzufügen von doppelten Einträgen in der Datenbank. Daher ist in der Praxis eine individuelle Implementierung erforderlich, um diese Herausforderungen zu bewältigen.

# Beispiel

In [HttpClientAutoConfiguration](https://github.com/ksewen/yorozuya/blob/main/yorozuya-spring-boot-starter/src/main/java/com/github/ksewen/yorozuya/starter/configuration/http/client/HttpClientAutoConfiguration.java) habe ich meiner Erfahrung nach die Konfiguration für HttpClient zusammengefasst.

Diskussionen, Fragen um Aufzeigen meiner Fehler sind willkommen.