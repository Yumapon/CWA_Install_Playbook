# Ansible Playbook: AgentInstallPlaybook

このPlaybookはCloudWatchAgentをインストールします。

* [参考資料](https://docs.aws.amazon.com/ja_jp/AmazonCloudWatch/latest/monitoring/install-CloudWatch-Agent-commandline-fleet.html)

## 対象OS

対象としているOSは以下の通りです。

* RedHat
  * 7
  * 8
* Windows
  * 2016
  * 2019

## 利用方法

* site.yml
* inventory/hosts
* roles/defaults/main.yaml
を編集し、playbookを実行してください。

roles/defaults/main.ymlで使用されている変数の説明は以下のとおり。

| 変数名 | 説明 | デフォルト値 | 設定可能な値 | 補足 |
|---------------|-------------------------------------------------------|------|----------------------------|-------------------------|
| use_metricset | AWSが用意しているメトリクスセットを利用するかどうかを指定できる | yes | yes,no | |
| cwa_metric_set | AWSが用意しているメトリクスセットを利用するかどうかを指定できる | Basic | Basic, Standard, Advanced | use_metricset にyesを設定した場合のみ指定可能|
| use_statsd | statdを利用したメトリクス収集を実施したい場合に使用 | false | false,true |  |
| use_collectd | collectdを利用したメトリクスを収集したい場合に使用 | false | false,true |  |
| conf_json_path | 自身で作成した設定ファイルを使用する | - | Playbook配下の任意のパス | 未実装 |
| logrotate_file_size | ローテーションされる前のログのファイルの最大サイズ | 10M | ※ |  |
| logrotate_files | ローテーション後に保持するファイルの最大数 | 5 | ※ |  |

※ [logrotateの設定ファイル](https://linux.die.net/man/8/logrotate)

## 利用例

ipが10.0.0.13のRHEL7(Passwordでsshログインできる状態)に対して、最小限のCloudWatchの設定を行う場合

* site.yml

    ```yaml
    ---
    - name: Agent構築
      hosts: targetaddress
      become: true
      strategy: debug
      gather_facts: yes
      roles:
       - roles
    ```

* inventory/hosts

    ```INI
    [targetaddress]
    10.0.0.13

    [targetaddress:vars]
    ansible_ssh_port=22
    ansible_user=ec2-user
    ansible_password=password
    ```

    ちなみにWIndowsだとこんな感じ。。

    ```INI
    [targetservers]
    10.0.1.137

    [targetservers:vars]
    ansible_user=Administrator
    ansible_password=ec2-user
    ansible_ssh_port=5985
    ansible_connection=winrm
    ansible_winrm_server_cert_validation=ignore
    ansible_become=yes
    ansible_become_method=runas
    ansible_become_user=SYSTEM
    ```

* roles/defaults/main.yaml(Linuxの一部機能とwindowsは未実装)

    ```yaml
    ---
    #---------------------------------
    # Windows Linux 両環境で設定必要
    #---------------------------------

    # エージェントの設定を保持する
    # リファレンス：https://docs.aws.amazon.com/ja_jp/AmazonCloudWatch/latest/monitoring/CloudWatch-Agent-Configuration-File-Details.html

    # メトリクスセットを使用して設定するか
    # 可能な値：
    # - yes
    # - no
    use_metricset: "yes"

    # 収集するメトリクスセットのタイプ
    # use_metricset: yes の場合に設定
    # 可能な値：
    # - Basic
    # - Standard
    # - Advanced
    # デフォルト値：
    # - Basic
    cwa_metric_set: "Basic"

    # StatsDを使用してメトリクスの収集を実施するか
    # use_metricset: yes の場合に設定
    # 可能な値：
    # - true
    # - false
    use_statsd: "false"

    # Collectdを使用してメトリクスの収集を実施するか
    # use_metricset: yes の場合に設定
    # 可能な値：
    # - true
    # - false
    use_collectd: "false"

    # 自身で作成した設定ファイルを使用する
    # use_metricset: noの場合に設定
    conf_json_path: ""

    #cwaで取得するログファイル
    collect_files:
    - path: /var/log/ansible.log
        name: ansible.log

    #---------------------------------
    # Linuxに使用する場合のみ設定必要
    #---------------------------------

    # 説明：ローテーションされる前のログのファイルの最大サイズ
    # 可能な値：(以下サイトを参照)
    # https://linux.die.net/man/8/logrotate
    # デフォルト値：「10M」
    logrotate_file_size: "10M"

    # 説明：ローテーション後に保持するファイルの最大数
    # 可能な値：
    # -https://linux.die.net/man/8/logrotate
    # デフォルト値：5
    logrotate_files: 5

    #---------------------------------
    #  Windowsに使用する場合のみ設定必要
    #---------------------------------
    ```

## 参考情報

* Windowsの設定ファイル

```json
{
    "logs": {
        "logs_collected": {
            "files": {
                "collect_list": [
                        {
                            "file_path": "C:\\Windows\\System32\\winevt\\Logs\\Windows PowerShell.evtx",
                            "log_group_name": "Windows PowerShell",
                            "log_stream_name": "{instance_id}"
                        }
                ]
                    },
            "windows_events": {
                "collect_list": [
                    {
                        "event_format": "text",
                        "event_levels": [
                            "VERBOSE",
                            "INFORMATION",
                            "WARNING",
                            "ERROR",
                            "CRITICAL"
                        ],
                        "event_name": "System",
                        "log_group_name": "System",
                        "log_stream_name": "{instance_id}"
                    }
                ]
            }
        }
    },
    "metrics": {
        "append_dimensions": {
            "AutoScalingGroupName": "${aws:AutoScalingGroupName}",
            "ImageId": "${aws:ImageId}",
            "InstanceId": "${aws:InstanceId}",
            "InstanceType": "${aws:InstanceType}"
        },
        "metrics_collected": {
            "LogicalDisk": {
                "measurement": [
                    "% Free Space"
                ],
                "metrics_collection_interval": 60,
                "resources": [
                    "*"
                ]
            },
            "Memory": {
                "measurement": [
                    "% Committed Bytes In Use"
                ],
                "metrics_collection_interval": 60
            }
        }
    }
}
```

* Basicで、StatDもCollectdも使用しないver(Linux)

    ```json
    {
        "agent": {
            "metrics_collection_interval": 60,
            "run_as_user": "root"
        },
        "metrics": {
            "append_dimensions": {
                "AutoScalingGroupName": "${aws:AutoScalingGroupName}",
                "ImageId": "${aws:ImageId}",
                "InstanceId": "${aws:InstanceId}",
                "InstanceType": "${aws:InstanceType}"
            },
            "metrics_collected": {
                "disk": {
                    "measurement": [
                        "used_percent"
                    ],
                    "metrics_collection_interval": 60,
                    "resources": [
                        "*"
                    ]
                },
                "mem": {
                    "measurement": [
                        "mem_used_percent"
                    ],
                    "metrics_collection_interval": 60
                }
            }
        }
    }
    ```

* Standardで、StatDもCollectdも使用しないver(Linux)

    ```json
    {
        "agent": {
            "metrics_collection_interval": 60,
            "run_as_user": "root"
        },
        "metrics": {
            "append_dimensions": {
                "AutoScalingGroupName": "${aws:AutoScalingGroupName}",
                "ImageId": "${aws:ImageId}",
                "InstanceId": "${aws:InstanceId}",
                "InstanceType": "${aws:InstanceType}"
            },
            "metrics_collected": {
                "cpu": {
                    "measurement": [
                        "cpu_usage_idle",
                        "cpu_usage_iowait",
                        "cpu_usage_user",
                        "cpu_usage_system"
                    ],
                    "metrics_collection_interval": 60,
                    "totalcpu": false
                },
                "disk": {
                    "measurement": [
                        "used_percent",
                        "inodes_free"
                    ],
                    "metrics_collection_interval": 60,
                    "resources": [
                        "*"
                    ]
                },
                "diskio": {
                    "measurement": [
                        "io_time"
                    ],
                    "metrics_collection_interval": 60,
                    "resources": [
                        "*"
                    ]
                },
                "mem": {
                    "measurement": [
                        "mem_used_percent"
                    ],
                    "metrics_collection_interval": 60
                },
                "swap": {
                    "measurement": [
                        "swap_used_percent"
                    ],
                    "metrics_collection_interval": 60
                }
            }
        }
    }

    ```

* Advancedで、StatDもCollectdも使用しないver(Linux)

    ```json
    {
        "agent": {
            "metrics_collection_interval": 60,
            "run_as_user": "root"
        },
        "metrics": {
            "append_dimensions": {
                "AutoScalingGroupName": "${aws:AutoScalingGroupName}",
                "ImageId": "${aws:ImageId}",
                "InstanceId": "${aws:InstanceId}",
                "InstanceType": "${aws:InstanceType}"
            },
            "metrics_collected": {
                "cpu": {
                    "measurement": [
                        "cpu_usage_idle",
                        "cpu_usage_iowait",
                        "cpu_usage_user",
                        "cpu_usage_system"
                    ],
                    "metrics_collection_interval": 60,
                    "totalcpu": false
                },
                "disk": {
                    "measurement": [
                        "used_percent",
                        "inodes_free"
                    ],
                    "metrics_collection_interval": 60,
                    "resources": [
                        "*"
                    ]
                },
                "diskio": {
                    "measurement": [
                        "io_time",
                        "write_bytes",
                        "read_bytes",
                        "writes",
                        "reads"
                    ],
                    "metrics_collection_interval": 60,
                    "resources": [
                        "*"
                    ]
                },
                "mem": {
                    "measurement": [
                        "mem_used_percent"
                    ],
                    "metrics_collection_interval": 60
                },
                "netstat": {
                    "measurement": [
                        "tcp_established",
                        "tcp_time_wait"
                    ],
                    "metrics_collection_interval": 60
                },
                "swap": {
                    "measurement": [
                        "swap_used_percent"
                    ],
                    "metrics_collection_interval": 60
                }
            }
        }
    }
    ```

* StatsDやcollectdを使用する場合、metrics_collectdに以下が入る(Linux)

    ```json
    "statsd": {
                    "metrics_aggregation_interval": 60,
                    "metrics_collection_interval": 10,
                    "service_address": ":8125"
                }
    ```

    ```json
    "collectd": {
                    "metrics_aggregation_interval": 60
                }
    ```

* windowsでcwaの設定反映

    ```powershell
    & "C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent-ctl.ps1" -a fetch-co
    nfig -m ec2 -s -c file:'C:\Program Files\Amazon\AmazonCloudWatchAgent\amazon-cloudwatch-agent.json'
    ```

* windowsでproductIDを調べる

    ```powershell
    Get-WmiObject Win32_Product | where {$_.Name -match "^Amazon.*"} | `
    >> foreach {Write-Host $_.Name, $_.Version, $_.IdentifyingNumber}
    ```

    productidは`5D3EF386-F834-49B7-9B99-3C59BC22FE08`

## 参考サイト

[Ansible Gather Factで収集できる情報](https://docs.ansible.com/ansible/2.9_ja/user_guide/playbooks_variables.html)