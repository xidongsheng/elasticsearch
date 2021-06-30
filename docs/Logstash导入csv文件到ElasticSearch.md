# 需求
计划写一系列ElasticSearch如何快速搭建小型搜索系统的文章，需要一些实际数据来展示ElasticSearch的搜索原理，选择使用kaggle公开数据集[Movies Dataset](https://www.kaggle.com/rounakbanik/the-movies-dataset?select=movies_metadata.csv)，由于是用来展示ElasticSearch的搜索原理，只保留id,name两列，使用Logstash将数据导入ElasticSearch。
# Logstash原理
 Logstash是一个开源的数据收集引擎，是ELK技术栈中的L。可以完成数据的ETL，方便的将数据导入ElasticSearch等。
 Logstash数据处理流程有三个主要Stage，inputs → filters → outputs.
 - Inputs从文本，数据库等读取数据进来，每条记录为一个event
 - filter对数据做处理转换等操作，如分隔，小写等。
 - output将event写入数据持久化层。

# 数据导入步骤详解
## 配置Logstash的Pipeline
Logstash的Pipeline通过配置文件来配置，movie-pipeline.yml
```
input {
  file {
    path => ["/Users/dxi/data/movies_metadata.csv"]  
    start_position => "beginning"
  }
}

filter {
  csv {
    separator => ","
    columns => ["id", "title"]
    skip_header => "true"
  }
  mutate {
    copy => {"id" => "[@metadata][_id]"}
    remove_field => ["@version"]
  }
}

output {
  elasticsearch {
      index => "movies_metadata"
      document_id => "%{[@metadata][_id]}"
      doc_as_upsert => true
      action => "update"
      hosts => "http://localhost:9200"
      manage_template => true
      template => "/Users/dxi/Documents/projects/elasticsearch/logstash/movie-template.json"
      template_name => "movie-template"
      template_overwrite => true
  }
}
```
## 定义数据在ElasticSearch中的数据类型
数据进入到ElasticSearch中需要明确数据类型，数据Analyzer等，需要定义ES index template

```
{
    "template": "movies*",
    "order": 0,
    "settings": {
      "index.max_result_window": "50000",
      "analysis": {
        "filter": {
          "word_joiner": {
            "type": "word_delimiter",
            "generate_word_parts": true,
            "generate_number_parts": true,
            "split_on_numerics": false,
            "split_on_case_change": false,
            "preserve_original": true,
            "catenate_all": true,
            "catenate_numbers": false,
            "catenate_words": false
          },
          "autocomplete": {
            "type": "edge_ngram",
            "min_gram": 2,
            "max_gram": 18,
            "token_chars": [
              "letter",
              "digit"
            ]
          }
        },
        "analyzer": {
          "title_analyzer": {
            "type": "custom",
            "tokenizer": "whitespace",
            "filter": [
              "lowercase",
              "word_joiner",
              "autocomplete"
            ]
          },
          "title_search_analyzer": {
            "type": "custom",
            "tokenizer": "whitespace",
            "filter": [
              "lowercase",
              "word_joiner"
            ]
          }
        }
      }
    },
    "mappings": {
      "properties": {
        "title": {
          "type": "text",
          "analyzer": "title_analyzer",
          "search_analyzer": "title_search_analyzer",
          "fields": {
            "raw": {
              "type": "keyword"
            }
          }
        },
        "release_date": {
          "type": "date"
        }
      }
    }
  }
```
## 数据导入
运行Logstash

```
logstash -f movie-pipeline.yml
```
Chrome中安装ElasticSearch Head插件，查看数据是否已经导入。
![在这里插入图片描述](https://img-blog.csdnimg.cn/20210624140601543.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3hkc2h1c3Q=,size_16,color_FFFFFF,t_70#pic_center)
## Logstash如何保证不重复导入
file input 插件会将文件状态写入到sincedb文件中，默认sincedb是存在data目录下。
```bash
Tiger:file dxi$ pwd
/usr/local/Cellar/logstash-full/7.13.2/libexec/data/plugins/inputs/file
Tiger:file dxi$ cat .sincedb_91f4e9a506f303fbd10bfcf80a2a84b4
21022558 1 4 1138439 1624507508.353749 /Users/dxi/data/movies_metadata.csv
```
sincedb中记录了文件的inode，时间戳，路径等。若Logstash重新启动首先会读写sincedb文件，确保从上次状态记录继续操作，而不会出现重复导入。
如果你确实需要重新导入，则删掉sincedb文件即可。
# 总结
Logstash已经有很多插件供使用，可以快速实现数据的ETL，不需要额外写代码。如果ETL逻辑特别复杂，filter支持用Ruby实现更加灵活的数据处理操作，Logstash是用JRuby写的，所以不需额外配置开发环境。
