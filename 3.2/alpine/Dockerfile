FROM mhart/alpine-node:6

MAINTAINER iyannik0215 , <admin@mashiro.io>

RUN sed -i 's/dl-cdn.alpinelinux.org/mirrors.ustc.edu.cn/g' /etc/apk/repositories \
    && apk --update --no-progress add git openssh

WORKDIR /Hexo

RUN \
    npm install hexo-cli -g --registry=https://registry.npm.taobao.org\
    && hexo init . \
    && npm install --registry=https://registry.npm.taobao.org\
    && npm install hexo-generator-sitemap --save --registry=https://registry.npm.taobao.org\
    && npm install hexo-generator-feed --save --registry=https://registry.npm.taobao.org\
    && npm install hexo-deployer-git --save --registry=https://registry.npm.taobao.org

#VOLUME ["/Hexo/source", "/Hexo/themes", "/root/.ssh"]

EXPOSE 80

COPY docker-entrypoint.sh /docker-entrypoint.sh

ENTRYPOINT ["/docker-entrypoint.sh"]

CMD ['/bin/bash']
