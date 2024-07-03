---
title:  "A100, A6000로 pytorch DP, DDP 학습시 멈추는 문제 해결 (ACS disable)"
search: false
categories: 
  - blog
  - bugs
last_modified_at: 2024-07-03T10:55:00+09:00
sidebar:
    nav: "sidebar-category"
---

A100, A6000 GPU로 pytorch의 DataParallel, DistributedDataParallel 학습시 프리징되는 현상을 경험했었다.

여러모로 찾아본 결과, 내 경우에는 ACS(Access Control Service)를 disable해야 했다.
ACS를 끄는 방법 두가지를 찾았는데, 첫째는 서버의 BIOS를 업데이트하는 것이다.
이 경우에는 직접 하기에는 전문성도 없고, 그렇다고 업체를 부르자니 시간이 걸리기에 포기했다.

두번째 방법은 직접 커맨드라인으로 ACS를 끄는 것이다.
이 경우, 서버를 재부팅할 때마다 다시 ACS를 일일히 꺼줘야 하는 불편함이 있다.
다만 ACS 끄는걸 bash script로 만들어서 부팅때 자동으로 실행하면 어느정도 해소되긴 한다.

나는 3단계에 걸쳐서 ACS 끄는 bash script를 만들었다.
먼저, 아래 커맨드를 이용해 lspci 결과를 저장한다.
``` bash
sudo lspci -vvvv > lspci_output.txt
```

둘째, 아래의 python script를 작성한다. 파일명은 `get_acs_enabled_bridges.py`로 설정했다.
``` python
# first, run 'sudo lspci -vvvv' and save as txt file
# then run this script to get sh file
import os
import argparse
import re

def parse_args():
    parser = argparse.ArgumentParser()

    parser.add_argument('--txt', help='txt file containing "sudo lspci -vvvv" output', type=str, default='')
    parser.add_argument('--out_path', help='output sh file', type=str, default='setpci.sh')
    args = parser.parse_args()
    return args

def main():
    args = parse_args()

    txt_path = args.txt
    with open(txt_path, 'r') as f:
        lines = f.readlines()

    hex_form = '[0-9a-fA-F]'
    bridge_id_p = re.compile(f'^{hex_form}+:{hex_form}+.{hex_form}+ ')
    acs_p1 = re.compile(f'^\t\tACSCtl:\tSrcValid+')
    acs_p2 = re.compile(f'^\t\tACSCtl: SrcValid+')

    n_line = len(lines)
    li = 0

    acs_ids = []
    while li+1 < n_line:
        line = lines[li]
        id_res = bridge_id_p.match(line)
        if id_res is not None:
            id_str = line[id_res.start():id_res.end()-1]
            next_li = li+1
            while (next_li < n_line):
                next_line = lines[next_li]
                if bridge_id_p.match(next_line) is not None:
                    break
                acs_res1 = acs_p1.match(next_line)
                acs_res2 = acs_p2.match(next_line)
                if (acs_res1 is not None) or (acs_res2 is not None):
                    acs_ids.append(id_str)
                    print(f"{next_li}: {next_line}")
                    break
                else:
                    next_li += 1

        li += 1

    if len(acs_ids)>0:
        out_path = args.out_path
        with open(out_path, 'w') as f:
            f.write('#!/bin/bash\n')
            for acs_id in acs_ids:
                                #print(f'sudo setpci -s {acs_id} f2a.w=0000')
                f.write(f'sudo setpci -s {acs_id} f2a.w=0000\n')
        print(f"{out_path} file written.")
    else:
        print("No acs enabled bus detected")

if __name__ == "__main__":
    main()
```

그 다음, 아래의 커맨드를 이용해 python 코드를 실행시켜 bash script 파일을 생성한다.
``` bash
python get_acs_enabled_bridges.py --txt lspci_output.txt --out_path setpci.sh
```

마지막으로 아래 커맨드로 bash script를 실행시켜 enable된 ACS를 모두 비활성화한다.
``` bash
chmod 777 setpci.sh
sudo ./setpci.sh
```
