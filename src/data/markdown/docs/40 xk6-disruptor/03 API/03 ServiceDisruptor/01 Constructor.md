---
title: 'Constructor'
excerpt: 'xk6-disruptor: ServiceDisruptor constructor'
---


The `ServiceDisruptor()` constructor creates a new instance of a [ServiceDisruptor](/javascript-api/xk6-disruptor/api/servicedisruptor) class.


| Parameter | Type | Description |
| --------- | ---- | ----------- |
| service   | string | name of the service |
| namespace | string | namespace on which the service is defined |
| options (optional) | object | [options](#options) for controlling the behavior of the disruptor |


### Options

The following options control the creation and behavior of the `ServiceDisruptor`:

| Attribute | Type | Description |
| --------- | ---- |------ |
| injectTimeout | string | maximum time for waiting the disruptor to be ready in the target pods (default 30s) |

## Example

<!-- eslint-skip -->

```javascript
  const disruptor = new ServiceDisruptor('my-service', 'my-namespace');
```
