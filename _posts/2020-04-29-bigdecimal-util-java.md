---
layout: post
title: "BigDecimal Utility Class Java"
permalink: "bigdecimal-util-java/"
last_modified_at: 2020-04-12T00:00:00
excerpt: "A simple utility class for BigDecimal Java"
category: "java"
---

A simple utility class for Java BigDecimal comparison

```java
import java.math.BigDecimal;

public class BigDecimalUtils {

    private BigDecimalUtils() {
    }

    public static boolean isEqualTo(BigDecimal arg0, BigDecimal arg1) {
        return arg0.compareTo(arg1) == 0;
    }

    public static boolean isGreaterThan(BigDecimal arg0, BigDecimal arg1) {
        return arg0.compareTo(arg1) > 0;
    }

    public static boolean isGreaterThanOrEqualTo(BigDecimal arg0, BigDecimal arg1) {
        return arg0.compareTo(arg1) >= 0;
    }

    public static boolean isLessThan(BigDecimal arg0, BigDecimal arg1) {
        return arg0.compareTo(arg1) < 0;
    }

    public static boolean isLessThanOrEqualTo(BigDecimal arg0, BigDecimal arg1) {
        return arg0.compareTo(arg1) <= 0;
    }

    public static boolean isNegative(BigDecimal arg0) {
        return arg0.compareTo(BigDecimal.ZERO) < 0;
    }

    public static boolean isNonNegative(BigDecimal arg0) {
        return arg0.compareTo(BigDecimal.ZERO) >= 0;
    }

    public static boolean isZero(BigDecimal arg0) {
        return arg0.compareTo(BigDecimal.ZERO) == 0;
    }
}
```