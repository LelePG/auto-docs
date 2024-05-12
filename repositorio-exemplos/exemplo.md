+++
title = '{{ (delimit (after 1 (split .File.ContentBaseName "-")) " ") | title }}'
date = {{ .Date }}
draft = false
weight = {{ index (split (strings.TrimPrefix "0" .File.ContentBaseName) "-") 0 | int }}
+++
