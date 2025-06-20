#!/bin/bash
# 文件名: auto_config_multi_ip.sh

# 检测操作系统类型
if [[ "$OSTYPE" == "linux-gnu"* ]]; then
    # 获取发行版信息
    if [ -f /etc/os-release ]; then
        . /etc/os-release
        OS=$ID
    else
        OS=$(uname -s)
    fi
    
    echo "检测到Linux系统: $OS"
    
    # 显示当前网络配置
    echo -e "\n当前网络接口状态:"
    ip -br addr show
    echo "--------------------------------"
    
    # 公共输入部分
    read -p "选择配置模式 (1=添加静态IP, 2=切换为DHCP, 3=管理现有IP): " mode
    
    # 获取非lo接口列表
    interfaces=($(ip -o link show | awk -F': ' '{print $2}' | grep -v lo))
    echo -e "\n可用的网络接口: ${interfaces[@]}"
    read -p "输入要配置的网卡名称: " iface
    
    # 验证接口是否存在
    if ! ip link show "$iface" &>/dev/null; then
        echo "错误：接口 $iface 不存在！"
        exit 1
    fi

    # CentOS/RHEL 配置
    if [[ "$OS" == "centos" || "$OS" == "rhel" || "$OS" == "fedora" ]]; then
        CONFIG_FILE="/etc/sysconfig/network-scripts/ifcfg-$iface"
        
        case $mode in
            1)  # 添加静态IP
                current_ips=($(ip -o -4 addr show $iface | awk '{print $4}'))
                echo -e "\n当前已配置IP:"
                printf '%s\n' "${current_ips[@]}"
                
                read -p "输入要添加的IP地址 (如192.168.1.100): " ip
                read -p "输入子网掩码 (如255.255.255.0): " netmask
                
                # 如果是第一个IP则要求网关和DNS
                if [ ${#current_ips[@]} -eq 0 ]; then
                    read -p "输入网关地址 (如192.168.1.1): " gateway
                    read -p "输入主DNS (如8.8.8.8): " dns1
                    read -p "输入备DNS (留空可跳过): " dns2
                fi
                
                # 添加临时IP (立即生效)
                sudo ip addr add $ip/$netmask dev $iface
                
                # 永久化配置
                if [ ! -f "$CONFIG_FILE" ]; then
                    sudo tee "$CONFIG_FILE" > /dev/null <<EOF
DEVICE=$iface
BOOTPROTO=none
ONBOOT=yes
EOF
                fi
                
                # 判断是主IP还是辅助IP
                if [ ${#current_ips[@]} -eq 0 ]; then
                    # 主IP配置
                    sudo tee -a "$CONFIG_FILE" > /dev/null <<EOF
IPADDR=$ip
NETMASK=$netmask
GATEWAY=$gateway
DNS1=$dns1
EOF
                    [ -n "$dns2" ] && echo "DNS2=$dns2" | sudo tee -a "$CONFIG_FILE" > /dev/null
                else
                    # 辅助IP配置
                    ip_count=$(grep -c "IPADDR" "$CONFIG_FILE")
                    sudo tee -a "$CONFIG_FILE" > /dev/null <<EOF
IPADDR$((ip_count+1))=$ip
NETMASK$((ip_count+1))=$netmask
EOF
                fi
                
                # 重启网络
                sudo systemctl restart network
                echo -e "\nIP $ip/$netmask 已成功添加!"
                ;;
                
            2)  # 切换为DHCP
                sudo tee "$CONFIG_FILE" > /dev/null <<EOF
DEVICE=$iface
BOOTPROTO=dhcp
ONBOOT=yes
EOF
                # 清除所有现有IP
                sudo ip addr flush dev $iface
                sudo systemctl restart network
                echo -e "\n已切换为DHCP模式!"
                ;;
                
            3)  # 管理现有IP
                current_ips=($(ip -o -4 addr show $iface | awk '{print $4}'))
                if [ ${#current_ips[@]} -eq 0 ]; then
                    echo "该接口没有配置任何IP地址"
                    exit 0
                fi
                
                echo -e "\n当前配置的IP:"
                select ip in "${current_ips[@]}" "删除所有IP" "取消"; do
                    case $ip in
                        "取消") exit 0 ;;
                        "删除所有IP")
                            sudo ip addr flush dev $iface
                            sudo rm -f "$CONFIG_FILE"
                            sudo systemctl restart network
                            echo "所有IP已删除!"
                            break
                            ;;
                        *)
                            sudo ip addr del $ip dev $iface
                            # 从配置文件中删除
                            ip_only=${ip%/*}
                            sudo sed -i "/IPADDR.*=$ip_only/d" "$CONFIG_FILE"
                            sudo sed -i "/NETMASK.*=/d" "$CONFIG_FILE"
                            sudo systemctl restart network
                            echo "IP $ip 已删除!"
                            break
                            ;;
                    esac
                done
                ;;
            *)
                echo "无效选择!"
                exit 1
                ;;
        esac
    
    # Ubuntu/Debian 配置
    elif [[ "$OS" == "ubuntu" || "$OS" == "debian" ]]; then
        config_file="/etc/netplan/01-netcfg.yaml"
        
        case $mode in
            1)  # 添加静态IP
                current_ips=($(ip -o -4 addr show $iface | awk '{print $4}'))
                echo -e "\n当前已配置IP:"
                printf '%s\n' "${current_ips[@]}"
                
                read -p "输入要添加的IP地址 (如192.168.1.100): " ip
                read -p "输入子网掩码前缀 (如24): " prefix
                
                # 如果是第一个IP则要求网关和DNS
                if [ ${#current_ips[@]} -eq 0 ]; then
                    read -p "输入网关地址 (如192.168.1.1): " gateway
                    read -p "输入主DNS (如8.8.8.8): " dns1
                    read -p "输入备DNS (留空可跳过): " dns2
                fi
                
                # 添加临时IP
                sudo ip addr add $ip/$prefix dev $iface
                
                # 备份原配置
                sudo cp "$config_file" "${config_file}.bak"
                
                # 处理多IP配置
                if [ -f "$config_file" ] && grep -q "$iface:" "$config_file"; then
                    # 已有配置，追加IP
                    sudo sed -i "/addresses:/ s/\]/, $ip\/$prefix\]/" "$config_file"
                else
                    # 新配置
                    sudo tee "$config_file" > /dev/null <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    $iface:
      addresses: [$ip/$prefix]
      routes:
        - to: default
          via: $gateway
      nameservers:
        addresses: [$dns1$( [ -n "$dns2" ] && echo ", $dns2" )]
EOF
                fi
                
                sudo netplan apply
                echo -e "\nIP $ip/$prefix 已成功添加!"
                ;;
                
            2)  # 切换为DHCP
                sudo cp "$config_file" "${config_file}.bak"
                sudo tee "$config_file" > /dev/null <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    $iface:
      dhcp4: yes
EOF
                sudo ip addr flush dev $iface
                sudo netplan apply
                echo -e "\n已切换为DHCP模式!"
                ;;
                
            3)  # 管理现有IP
                current_ips=($(ip -o -4 addr show $iface | awk '{print $4}'))
                if [ ${#current_ips[@]} -eq 0 ]; then
                    echo "该接口没有配置任何IP地址"
                    exit 0
                fi
                
                echo -e "\n当前配置的IP:"
                select ip in "${current_ips[@]}" "删除所有IP" "取消"; do
                    case $ip in
                        "取消") exit 0 ;;
                        "删除所有IP")
                            sudo ip addr flush dev $iface
                            sudo tee "$config_file" > /dev/null <<EOF
network:
  version: 2
  renderer: networkd
  ethernets:
    $iface:
      dhcp4: yes
EOF
                            sudo netplan apply
                            echo "所有IP已删除!"
                            break
                            ;;
                        *)
                            sudo ip addr del $ip dev $iface
                            ip_only=${ip%/*}
                            sudo sed -i "/addresses:/ s/\(.*\)$ip\/[0-9]*\(.*\)/\1\2/" "$config_file"
                            sudo sed -i "/addresses:/ s/,\s*\]/\]/" "$config_file"
                            sudo sed -i "/addresses:/ s/\[\s*,/\[/" "$config_file"
                            sudo netplan apply
                            echo "IP $ip 已删除!"
                            break
                            ;;
                    esac
                done
                ;;
            *)
                echo "无效选择!"
                exit 1
                ;;
        esac
    else
        echo "不支持的Linux发行版: $OS"
        exit 1
    fi

    # 显示配置后状态
    echo -e "\n--------------------------------"
    echo "新的网络接口状态:"
    ip -br addr show $iface
else
    echo "不支持的操作系统: $OSTYPE"
    exit 1
fi
