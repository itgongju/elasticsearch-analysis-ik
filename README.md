1. 扩展词典
IK 提供了配置文件 IKAnalyzer.cfg.xml，可以用来配置自己的扩展词典和远程扩展词典，都可以配置多个。

配置完扩展词典和远程扩展词典都需要重启ES，后续对词典进行更新的话，扩展词典的话需要重启ES，远程扩展词典配置完后支持热更新，每60秒检查更新。两个扩展词典都是添加到 IK 的主词典中，对所有索引生效。

<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE properties SYSTEM "http://java.sun.com/dtd/properties.dtd">
<properties>
	<comment>IK Analyzer 扩展配置</comment>
	<!--用户可以在这里配置自己的扩展字典 -->
	<entry key="ext_dict">my_ik.dic</entry>
	 <!--用户可以在这里配置自己的扩展停止词字典-->
	<entry key="ext_stopwords"></entry>
	<!--用户可以在这里配置远程扩展字典 -->
	<entry key="remote_ext_dict">http://localhost:8000/dic/remote.dic</entry>
	<!--用户可以在这里配置远程扩展停止词字典-->
	<!-- <entry key="remote_ext_stopwords">words_location</entry> -->
</properties>

远程扩展词典可使用 Nginx，修改 conf/nginx.conf 配置，再在 Nginx 安装目录下创建文件夹 dic，在里面放置远程词典。

server {
    listen       8000;
    server_name  localhost;
    
    # IK 远程扩展词典
    location /dic {
        alias dic;
    }
}

2. 隔离词典
IK 有个词典类 Dictionary，默认有三个词典：主词典 _MainDict、量词词典 _QuantifierDict 和停顿词典 _StopWords。在分词时，主要通过主词典进行分词匹配，分完的词在停顿词典中，就抛弃。IK 默认是在配置的时候，就初始化该词典类了，将词典文件加载到内存中。如果配置了远程词典，会在线程池中创建检查更新的任务，实现词典的热更新。远程扩展词典添加到主词典，远程扩展停顿词典添加到停顿词典。

在 ES 中使用时，IK 使用的都是同一个词典类，即默认那三个词典。如果要实现自定义分词，可以使用远程词典，远程词典都会加到原词典中，对所有的索引都生效。如果要实现索引间不同的分词需求，就需要对词库进行隔离。

所以可以在词典类 Dictionary 中可以添加一个新的词典，考虑到索引间不同的分词需求，存在词典间隔离，使用 Map 存储不同的词典。在创建索引的配置中，添加配置项，指定需要使用的词典。再创建相应的检查更新的任务，实现热更新。在分词时，在主词典的匹配方法中，前置添加对新词典的匹配，实现干预。

修改后源码见 GitHub shpunishment/elasticsearch-analysis-ik

在定义分词器时设置词典url来获取词典，实现不同索引使用不同的词典，然后在查询时进行干预。

PUT ik_test_1
{
  "settings": {
    "analysis": {
      "analyzer": {
        "my_ik_analyzer": {
          "tokenizer": "my_ik_tokenizer"
        }
      },
      "tokenizer": {
        "my_ik_tokenizer": {
          "type": "ik_max_word",
          "dict_url": "http://localhost:8123/dict/1"
        }
      }
    }
  }
}
