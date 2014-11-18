# Anomalyzer

[![Build Status](https://travis-ci.org/lytics/anomalyzer.svg?branch=master)](https://travis-ci.org/lytics/anomalyzer) [![GoDoc](https://godoc.org/github.com/lytics/anomalyzer?status.svg)](https://godoc.org/github.com/lytics/anomalyzer)

Probability-based anomaly detection in Go.

## Event Windows

Anomalyzer implements a suite of statistical tests that yield the probability that a given set of numeric input, typically a time series, contains anomalous behavior. It can be used in batch or streaming. Inspired by Etsy's [Skyline](https://github.com/etsy/skyline) package.

Anomalyzer works against a **reference window** of events and an **active window** of events, it does not have a notion of "wall clock time", it's only sense of time is new events arriving.

```
    | e0  e1  e2  e3  e4  e5 | e6  e7  e8  e9 |
    |------------------------|----------------|-----> time, newer events
    |   Reference   Window   | Active  Window |
```

The **active window** is set directly in configuration `AnomalyzerConf{ ActiveSize: 100 }`. The **reference window** is set implicitly by choosing the "number of seasons," for example: `AnomalyzerConf{ NSeasons: 10 }` would set the **reference window** to 1000 events, in other words the size of 10 active windows. If you don't specify the number of seasons it defaults to 4.

## Example Batch

```go
package main

import (
	"fmt"
	"github.com/lytics/anomalyzer"
)

func main() {
	conf := &anomalyzer.AnomalyzerConf{
		ActiveSize:  2,
		NSeasons:    3,                 // Reference window is implicitly set to 6.
		UpperBound:  5,                 // Upper bound.
		LowerBound:  anomalyzer.NA,     // Ignore the lower bound.
		Methods:     []string{"fence"}, // But more than one can be used.
	}

	// Initialize with reference data. The last two values
	// would be considered the "active window" since its
	// size was set to 2, the first six values would be
	// considered the "reference window" since the number
	// of seasons was set to 3, ie: season size is the
	// same as the active window size.
	data := []float64{
		0.1, 2.05,
		1.5, 2.5, 
		2.6, 2.55, 
		1.5, 1.1,
	}

	anom, err := anomalyzer.NewAnomalyzer(conf, data)
	if err != nil {
		fmt.Printf("error: failed to create anomalyzer: %v\n", err)
		return
	}

	// Eval calculates the probability of anomaly in the
	// currently data.
	p := anom.Eval()
	fmt.Println("anomalous probability:", p)
}
```

## Example Streaming

```go
package main

import (
	"fmt"
	"github.com/lytics/anomalyzer"
)

func openstream() (<-chan float64, error) {
	... open stream ...
}

func main() {
	conf := &anomalyzer.AnomalyzerConf{
		ActiveSize:  100,
		NSeasons:    4,               // Reference window set to 400.
		Methods:     []string{"cdf"}, // Use just one test.
	}

	anom, err := anomalyzer.NewAnomalyzer(conf, nil)
	if err != nil {
		fmt.Printf("error: failed to create anomalyzer: %v\n", err)
		return
	}

	stream, err := openstream()
	if err != nil {
		fmt.Printf("error: failed to create stream: %v\n", err)
	}

	for e := range stream {
		// Push automatically recalculates the
		// anomaly probability of the pushed
		// event.
		p := anom.Push(e)
		fmt.Printf("anomalous probability: %v\n", p)
	}
}
```

## Algorithms & Configuration

Anomalyzer can be set to use multiple detectors at the same time. For example, by setting `AnomalyzerConf{ Methods: []string{"cdf", "fence" } }` both algorithms would be used. 

Each test yields a probability of anomalous behavior, and the probabilities are then computed over a weighted mean to determine if the overall behavior is anomalous. Since a *probability* is returned, the user can determine the threshold for anomalous behavior for the application, whether at say 0.8 for general anomalous behavior or 0.95 for extreme anomalous behavior.


### Choosing

1. **cdf** is suitable for detecting anomalous increases or decreases in the data. Theoretically, you could use it instead of using both high rank and low rank. It tends to be very sensitive.

2. **diff** is suitable for detecting anomalous changes in volatile data, like fine CPU samples. It's not the ideal test for steadily increasing or decreasing data.

3. **high rank** is suitable for detecting anomalous increases in the data. It can detect gradual change better than diff or magnitude might.

4. **low rank** is suitable for detecting anomalous decreases in the data. It can detect gradual change better than diff or magnitude might.

5. **fence** is suitable when anomalies constitute trending to some upper or lower bound. Use the configuration `AnomalyzerConf{ LowerBound: lower }` and `AnomalyzerConf{ UpperBound: upper }` to set the bounds. The value `anomalyzer.NA` can be used to have no lower bound.

6. **magnitude** is suitable for detecting if event values have changed sufficiently to be considered an anomaly. Use the configuration `AnomalyzerConf{ Sensitivity: 0.1 }` to require that values have changed at least 10% to consititue an anomaly. In general, sensitivity can be set between 0 and 1. If the result of the magnitude test is less than that value, the weighted mean will return 0. If `Sensitivity` is not specified, it defaults to 0.1.

7. **bootstrap ks** is especially suitable if your data is seasonal. For example, if you want to check a days worth of data and compare that to the previous four, then `ActiveSize` should be set to the number of events received in a day. And `NSeasons` should be set to 4. It calculates the [Kolmogorov-Smirnov](http://en.wikipedia.org/wiki/Kolmogorov%E2%80%93Smirnov_test) test over active and reference windows and compares that value to KS test scores obtained after permuting all elements in the set. 


### Diff, Bootstrap and Rank

The diff, bootstrap ks, and rank tests can accept a value for the number of bootstrap samples to generate, indicated by `PermCount`, and defaults to 500 if not set.
