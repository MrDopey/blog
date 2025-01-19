--- 
date: {{ .Date }}
draft: true
title: '{{ replace (replaceRE `^(\d{4}-\d{2})-` "" .File.ContentBaseName 1) "-" " " | title }}'
description: ""
slug: '{{ replaceRE `^(\d{4}-\d{2})-` "" .File.ContentBaseName 1 }}'
authors: []
tags: []
categories: []
externalLink: ""
series: []
---
