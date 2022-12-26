---
title: QLUT镜像站的日志监控及预警方案
cover: https://s2.loli.net/2022/12/24/FI1YkGehQyVKs9T.jpg
tags: Devops
date: 2022/12/07 1:27:20
---

十分意外镜像站用户增长速度如此之快，面对日均五万左右的用户量，服务器频繁出现各种性能问题。在故障排查的过程中，我意识到服务器需要一套成熟、完整、可靠的日志监控及预警方案。

先前长期使用 `goaccess` 静态分析Nginx的日志，具有滞后性。只能用于单纯的分析总结一段时间内的网站流量状况，而无法做到实时监看服务状态和运行数据。

既然是开源镜像站，那么我们就需要把目光投向开源社区。

## 较为流行的ELK

目前市面上最流行的日志处理方案莫过于“ELK”系统，“ELK”是Elasticsearch、Logstash和Kibana的组合的合称。

Elasticsearch 是一个搜索和分析引擎。Logstash 是服务器端数据处理管道，能够同时从多个来源采集数据，转换数据，然后将数据发送到诸如 Elasticsearch 等“存储库”中。Kibana 则可以让用户在 Elasticsearch 中使用图形和图表对数据进行可视化。

## InfluxDB&Telegraf

通过自己的初步了解以及咨询使用者的使用体验，我认为ELK在使用体验和性能上都不适合我（其实是单纯的不喜欢Java）。而一个朋友推荐我去尝试一下 InfluxDB+Telegraf 的组合。

服务器的性能日志从本质上来说，属于**时序大数据**，我们打开数据库排名网站[DB-Engines Ranking](https://db-engines.com/en/ranking)，左侧选择时序数据库([Time Series DBMS](https://db-engines.com/en/ranking/time+series+dbms))，可以看到 InfluxDB 在排名上一骑绝尘。

![image.png](https://s2.loli.net/2022/12/24/e8RbQm7i2Jj1Kzk.png)

安装直接跟着官方文档即可，下面我将英文文档翻译一下给大家参考：

#### 使用 systemd 将 InfluxDB 安装为服务

1. 使用以下命令从[InfluxData 下载页面](https://portal.influxdata.com/downloads/)使用 URL 下载并安装适当的`.deb`或文件：`.rpm`

   ```sh
   # Ubuntu/Debian
   wget https://dl.influxdata.com/influxdb/releases/influxdb2-2.6.0-xxx.deb
   sudo dpkg -i influxdb2-2.6.0-xxx.deb
   
   # Red Hat/CentOS/Fedora
   wget https://dl.influxdata.com/influxdb/releases/influxdb2-2.6.0-xxx.rpm
   sudo yum localinstall influxdb2-2.6.0-xxx.rpm
   ```

   *使用`.rpm`包下载的确切文件名（例如，`influxdb2-2.6.0-amd64.rpm`）。*

2. 启动 InfluxDB 服务：

   ```sh
   sudo service influxdb start
   ```

   安装 InfluxDB 包会创建一个服务文件，`/lib/systemd/system/influxdb.service` 用于在启动时将 InfluxDB 作为后台服务启动。

3. 重新启动系统并验证服务是否正常运行：

   ```shell
   $  sudo service influxdb status
   ● influxdb.service - InfluxDB is an open-source, distributed, time series database
     Loaded: loaded (/lib/systemd/system/influxdb.service; enabled; vendor preset: enable>
     Active: active (running)
   ```

有关 InfluxDB 在作为服务运行时在磁盘上存储数据的位置的信息，请参阅[文件系统布局](https://docs.influxdata.com/influxdb/v2.6/reference/internals/file-system-layout/?t=Linux#installed-as-a-package)。

要自定义 InfluxDB 配置，请使用 [命令行标志（参数）](https://docs.influxdata.com/influxdb/v2.6/install/?t=Linux#pass-arguments-to-systemd)、环境变量或 InfluxDB 配置文件。有关详细信息，请参阅 InfluxDB[配置选项](https://docs.influxdata.com/influxdb/v2.6/reference/config-options/)。

#### 将参数传递给 systemd

1. 添加一行或多行命令，如下所示，包含`influxd`to的参数`/etc/default/influxdb2`：

   ```sh
   ARG1="--http-bind-address :8087"
   ARG2="<another argument here>"
   ```

2. 编辑`/lib/systemd/system/influxdb.service`文件如下：

   ```sh
   ExecStart=/usr/bin/influxd $ARG1 $ARG2
   ```

#### 启动 InfluxDB

如果 InfluxDB 作为 systemd 服务安装，则 systemd 管理`influxd`守护进程，不需要进一步的操作。如果二进制文件是手动下载并添加到系统中的，请使用以下命令`$PATH`启动守护进程：

```shell
influxd
```

进入8086端口配置好后，就可以在DashBoard里查阅可视化数据了。

![image.png](https://s2.loli.net/2022/12/24/2ybGTCxF6znZQJA.png)

## Grafana

InfluxDB毕竟是一个数据库，在可视化方便并不擅长。因此我在另外一台机器上调用InfluxDB的数据源制作可视化面板。

![image.png](https://s2.loli.net/2022/12/26/Jeig5pqMROmwAbE.png)

面板主要包含了Nginx启动状态、内存利用、储存利用、CPU、DiskIO、网络状态和缓存命中率等。

下面是面板的JSON以供参考

```JSON
{
  "annotations": {
    "list": [
      {
        "builtIn": 1,
        "datasource": {
          "type": "grafana",
          "uid": "-- Grafana --"
        },
        "enable": true,
        "hide": true,
        "iconColor": "rgba(0, 211, 255, 1)",
        "name": "Annotations & Alerts",
        "target": {
          "limit": 100,
          "matchAny": false,
          "tags": [],
          "type": "dashboard"
        },
        "type": "dashboard"
      }
    ]
  },
  "editable": true,
  "fiscalYearStartMonth": 0,
  "graphTooltip": 0,
  "id": 1,
  "links": [],
  "liveNow": false,
  "panels": [
    {
      "datasource": {
        "type": "influxdb",
        "uid": "8-KvbZO4k"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [
            {
              "options": {
                "0": {
                  "index": 0,
                  "text": "DOWN"
                }
              },
              "type": "value"
            },
            {
              "options": {
                "from": 1,
                "result": {
                  "index": 1,
                  "text": "UP"
                },
                "to": 9999999
              },
              "type": "range"
            }
          ],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 3,
        "x": 0,
        "y": 0
      },
      "id": 21,
      "options": {
        "colorMode": "value",
        "graphMode": "area",
        "justifyMode": "auto",
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "textMode": "auto"
      },
      "pluginVersion": "9.2.4",
      "targets": [
        {
          "datasource": {
            "type": "influxdb",
            "uid": "8-KvbZO4k"
          },
          "query": "from(bucket: \"qlu\")\r\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\r\n  |> filter(fn: (r) => r[\"_measurement\"] == \"nginx\")\r\n  |> filter(fn: (r) => r[\"_field\"] == \"active\")\r\n  |> filter(fn: (r) => r[\"host\"] == \"i-30g60b69\")\r\n  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)\r\n  |> yield(name: \"mean\")",
          "refId": "A"
        }
      ],
      "title": "Nginx UP/DOWN",
      "type": "stat"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "8-KvbZO4k"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            }
          },
          "mappings": [],
          "min": 0,
          "unit": "decbytes"
        },
        "overrides": [
          {
            "matcher": {
              "id": "byName",
              "options": "cached i-30g60b69"
            },
            "properties": [
              {
                "id": "color",
                "value": {
                  "fixedColor": "light-blue",
                  "mode": "fixed"
                }
              }
            ]
          },
          {
            "matcher": {
              "id": "byName",
              "options": "free i-30g60b69"
            },
            "properties": [
              {
                "id": "color",
                "value": {
                  "fixedColor": "light-green",
                  "mode": "fixed"
                }
              }
            ]
          },
          {
            "matcher": {
              "id": "byName",
              "options": "used i-30g60b69"
            },
            "properties": [
              {
                "id": "color",
                "value": {
                  "fixedColor": "red",
                  "mode": "fixed"
                }
              }
            ]
          },
          {
            "matcher": {
              "id": "byName",
              "options": "buffered i-30g60b69"
            },
            "properties": [
              {
                "id": "color",
                "value": {
                  "fixedColor": "yellow",
                  "mode": "fixed"
                }
              }
            ]
          }
        ]
      },
      "gridPos": {
        "h": 8,
        "w": 6,
        "x": 3,
        "y": 0
      },
      "id": 10,
      "options": {
        "displayLabels": [
          "value"
        ],
        "legend": {
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true,
          "values": [
            "value"
          ]
        },
        "pieType": "donut",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "tooltip": {
          "mode": "single",
          "sort": "desc"
        }
      },
      "pluginVersion": "9.2.4",
      "targets": [
        {
          "datasource": {
            "type": "influxdb",
            "uid": "8-KvbZO4k"
          },
          "query": "from(bucket: \"qlu\")\r\n  |> range(start: v.timeRangeStart)\r\n  |> filter(fn: (r) => r._measurement == \"mem\")\r\n  |> filter(fn: (r) => r._field == \"used\" or r._field == \"cached\" or r._field == \"free\" or r._field == \"buffered\")\r\n  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)\r\n  |> yield(name: \"mean\")",
          "refId": "A"
        }
      ],
      "title": "Memory Usage",
      "transformations": [],
      "type": "piechart"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "8-KvbZO4k"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "thresholds"
          },
          "mappings": [],
          "max": 100,
          "min": 0,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          },
          "unit": "percent"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 3,
        "x": 9,
        "y": 0
      },
      "id": 12,
      "options": {
        "orientation": "auto",
        "reduceOptions": {
          "calcs": [
            "lastNotNull"
          ],
          "fields": "",
          "values": false
        },
        "showThresholdLabels": false,
        "showThresholdMarkers": true
      },
      "pluginVersion": "9.2.4",
      "targets": [
        {
          "datasource": {
            "type": "influxdb",
            "uid": "8-KvbZO4k"
          },
          "query": "from(bucket: \"qlu\")\r\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\r\n  |> filter(fn: (r) => r[\"_measurement\"] == \"disk\")\r\n  |> filter(fn: (r) => r[\"_field\"] == \"used_percent\")\r\n  |> filter(fn: (r) => r[\"device\"] == \"vdc\")\r\n  |> filter(fn: (r) => r[\"fstype\"] == \"ext4\")\r\n  |> filter(fn: (r) => r[\"host\"] == \"i-30g60b69\")\r\n  |> filter(fn: (r) => r[\"mode\"] == \"rw\")\r\n  |> filter(fn: (r) => r[\"path\"] == \"/mnt/vos-6izqwyw8\")\r\n  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)\r\n  |> yield(name: \"mean\")",
          "refId": "A"
        }
      ],
      "title": "Disk Usage",
      "type": "gauge"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "8-KvbZO4k"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 31,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "normal"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 0
      },
      "id": 19,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "single",
          "sort": "none"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "influxdb",
            "uid": "8-KvbZO4k"
          },
          "query": "from(bucket: \"qlu\")\r\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\r\n  |> filter(fn: (r) => r[\"_measurement\"] == \"nginx\")\r\n  |> filter(fn: (r) => r[\"_field\"] == \"waiting\" or r[\"_field\"] == \"writing\")\r\n  |> filter(fn: (r) => r[\"host\"] == \"i-30g60b69\")\r\n  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)\r\n  |> yield(name: \"mean\")",
          "refId": "A"
        }
      ],
      "title": "TCP Socket",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "8-KvbZO4k"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 79,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "lineInterpolation": "linear",
            "lineStyle": {
              "fill": "solid"
            },
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "normal"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          }
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 6,
        "x": 0,
        "y": 8
      },
      "id": 18,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "single",
          "sort": "none"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "influxdb",
            "uid": "8-KvbZO4k"
          },
          "query": "from(bucket: \"qlu\")\r\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\r\n  |> filter(fn: (r) => r[\"_measurement\"] == \"cpu\")\r\n  |> filter(fn: (r) => r[\"_field\"] == \"usage_iowait\" or r[\"_field\"] == \"usage_irq\" or r[\"_field\"] == \"usage_steal\" or r[\"_field\"] == \"usage_user\" or r[\"_field\"] == \"usage_system\"  or r[\"_field\"] == \"usage_softirq\" or r[\"_field\"] == \"usage_nice\" or r[\"_field\"] == \"usage_guest_nice\" or r[\"_field\"] == \"usage_guest\")\r\n  |> filter(fn: (r) => r[\"host\"] == \"i-30g60b69\")\r\n  |> filter(fn: (r) => r[\"cpu\"] == \"cpu-total\")\r\n  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)\r\n  |> yield(name: \"mean\")",
          "refId": "A"
        }
      ],
      "title": "CPU*",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "8-KvbZO4k"
      },
      "description": "",
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisGridShow": false,
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 50,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "linearThreshold": 1,
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "min": 0,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          },
          "unit": "Bps"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 6,
        "x": 6,
        "y": 8
      },
      "id": 6,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "single",
          "sort": "none"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "influxdb",
            "uid": "8-KvbZO4k"
          },
          "query": "from(bucket: \"qlu\")\r\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\r\n  |> filter(fn: (r) => r[\"_measurement\"] == \"diskio\")\r\n  |> filter(fn: (r) => r[\"_field\"] == \"read_bytes\" or r[\"_field\"] == \"write_bytes\")\r\n  |> filter(fn: (r) => r[\"host\"] == \"i-30g60b69\")\r\n  |> filter(fn: (r) => r[\"name\"] == \"vdc\")\r\n  |> derivative(unit: 1s, nonNegative: false)\r\n  |> yield(name: \"derivative\")",
          "refId": "A"
        }
      ],
      "title": "Disk IO",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "8-KvbZO4k"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 21,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "min": 0,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          },
          "unit": "bytes"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 8
      },
      "id": 14,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "single",
          "sort": "none"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "influxdb",
            "uid": "8-KvbZO4k"
          },
          "query": "from(bucket: \"qlu\")\r\n  |> range(start: v.timeRangeStart)\r\n  |> filter(fn: (r) => r._measurement == \"net\")\r\n  |> filter(fn: (r) => r._field == \"bytes_recv\" or r._field == \"bytes_sent\")\r\n  |> aggregateWindow(every: v.windowPeriod, fn: last, createEmpty: false)\r\n  |> derivative(unit: 1s, nonNegative: false)\r\n  |> yield(name: \"derivative\")",
          "refId": "A"
        }
      ],
      "title": "IPv4 Traffic",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "8-KvbZO4k"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": 0,
            "drawStyle": "line",
            "fillOpacity": 50,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "lineInterpolation": "linear",
            "lineWidth": 1,
            "pointSize": 5,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          },
          "unit": "decbytes"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 0,
        "y": 16
      },
      "id": 23,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "single",
          "sort": "none"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "influxdb",
            "uid": "8-KvbZO4k"
          },
          "query": "from(bucket: \"qlu\")\r\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\r\n  |> filter(fn: (r) => r[\"_measurement\"] == \"mem\")\r\n  |> filter(fn: (r) => r[\"_field\"] == \"cached\")\r\n  |> aggregateWindow(every: v.windowPeriod, fn: mean, createEmpty: false)\r\n  |> yield(name: \"mean\")",
          "refId": "A"
        }
      ],
      "title": "Memory Cached",
      "type": "timeseries"
    },
    {
      "datasource": {
        "type": "influxdb",
        "uid": "8-KvbZO4k"
      },
      "fieldConfig": {
        "defaults": {
          "color": {
            "mode": "palette-classic"
          },
          "custom": {
            "axisCenteredZero": false,
            "axisColorMode": "text",
            "axisLabel": "",
            "axisPlacement": "auto",
            "barAlignment": -1,
            "drawStyle": "bars",
            "fillOpacity": 86,
            "gradientMode": "none",
            "hideFrom": {
              "legend": false,
              "tooltip": false,
              "viz": false
            },
            "lineInterpolation": "smooth",
            "lineStyle": {
              "fill": "solid"
            },
            "lineWidth": 0,
            "pointSize": 4,
            "scaleDistribution": {
              "type": "linear"
            },
            "showPoints": "auto",
            "spanNulls": false,
            "stacking": {
              "group": "A",
              "mode": "none"
            },
            "thresholdsStyle": {
              "mode": "off"
            }
          },
          "mappings": [],
          "min": 0,
          "thresholds": {
            "mode": "absolute",
            "steps": [
              {
                "color": "green",
                "value": null
              },
              {
                "color": "red",
                "value": 80
              }
            ]
          },
          "unit": "percentunit"
        },
        "overrides": []
      },
      "gridPos": {
        "h": 8,
        "w": 12,
        "x": 12,
        "y": 16
      },
      "id": 25,
      "options": {
        "legend": {
          "calcs": [],
          "displayMode": "list",
          "placement": "bottom",
          "showLegend": true
        },
        "tooltip": {
          "mode": "single",
          "sort": "none"
        }
      },
      "targets": [
        {
          "datasource": {
            "type": "influxdb",
            "uid": "8-KvbZO4k"
          },
          "query": "from(bucket: \"qlu\")\r\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\r\n  |> filter(fn: (r) => r[\"_measurement\"] == \"diskio\")\r\n  |> filter(fn: (r) => r[\"_field\"] == \"read_bytes\")\r\n  |> filter(fn: (r) => r[\"host\"] == \"i-30g60b69\")\r\n  |> filter(fn: (r) => r[\"name\"] == \"vdc\")\r\n  |> derivative(unit: 10s, nonNegative: false)\r\n  |> yield(name: \"derivative\")",
          "refId": "A"
        },
        {
          "datasource": {
            "type": "influxdb",
            "uid": "8-KvbZO4k"
          },
          "hide": false,
          "query": "from(bucket: \"qlu\")\r\n  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)\r\n  |> filter(fn: (r) => r[\"_measurement\"] == \"net\")\r\n  |> filter(fn: (r) => r[\"_field\"] == \"bytes_sent\")\r\n  |> filter(fn: (r) => r[\"host\"] == \"i-30g60b69\")\r\n  |> filter(fn: (r) => r[\"interface\"] == \"eth0\")\r\n  |> derivative(unit: 10s, nonNegative: false)\r\n  |> yield(name: \"derivative\")",
          "refId": "B"
        }
      ],
      "title": "Cache Hit Rate",
      "transformations": [
        {
          "id": "calculateField",
          "options": {
            "alias": "REC",
            "binary": {
              "left": "bytes_sent {host=\"i-30g60b69\", interface=\"eth0\", name=\"net\"}",
              "operator": "-",
              "reducer": "sum",
              "right": "read_bytes {host=\"i-30g60b69\", name=\"diskio\"}"
            },
            "mode": "binary",
            "reduce": {
              "reducer": "sum"
            },
            "replaceFields": false
          }
        },
        {
          "id": "calculateField",
          "options": {
            "alias": "Cache Hit Rate",
            "binary": {
              "left": "REC",
              "operator": "/",
              "reducer": "sum",
              "right": "bytes_sent {host=\"i-30g60b69\", interface=\"eth0\", name=\"net\"}"
            },
            "mode": "binary",
            "reduce": {
              "reducer": "sum"
            },
            "replaceFields": true
          }
        }
      ],
      "type": "timeseries"
    }
  ],
  "refresh": "5s",
  "schemaVersion": 37,
  "style": "dark",
  "tags": [],
  "templating": {
    "list": []
  },
  "time": {
    "from": "now-6h",
    "to": "now"
  },
  "timepicker": {},
  "timezone": "",
  "title": "QLU_Mirrors",
  "uid": "z38bN0O4z",
  "version": 15,
  "weekStart": ""
}
```