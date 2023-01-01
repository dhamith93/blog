---
title: "Table_visualizer Visualize Database Table Connections"
date: 2021-11-19T21:03:15+05:30
draft: false
---

This is a tool to visualize all or selected table foreign-key connections of a database. Currently table_visualizer supports MySQL/MariaDB and PostgreSQL databases.

Download: https://github.com/dhamith93/table_visualizer/releases

## Usage

Visualize full table connections

Command:
```bash
./table_visualizer -u root -p password -h 172.17.0.2:3306 -d db_name -s mariadb|mysql|postgres -o filename.jpg

# -u : user name

# -p : password

# -h : host:port

# -d : db name

# -s : database type [ mysql | mariadb | postgres ]

# -o : output file name (jpg)
```

Example output

![](/table_vis_1.jpg)

Visualize connections of a single table

```bash
# -t : table name
```

With this option, table_visualizer will generate all foreign-key connections from a curtain table.

```bash
./table_visualizer -u root -p password -h 172.17.0.2:3306 -d db_name -t table_name -s mariadb|mysql|postgres -o filename.jpg
```

Example output

![](/table_vis_2.jpg)

## Compiling

Install Go https://golang.org/doc/install

Clone table_visualizer and cd into it https://github.com/dhamith93/table_visualizer

Run bellow command

```bash
make clean && make build
```

