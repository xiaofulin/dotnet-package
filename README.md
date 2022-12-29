#!/bin/bash

flag=`date '+%Y%m%d%H%M'`
comitid=`git log |grep commit |awk '{print $2}' | head -n 1`
chmod 777 -R ./*
write_file(){
        cat > ${WORKSPACE}/Dockerfile-manager <<EOF
FROM mcr.microsoft.com/dotnet/aspnet:5.0-buster-slim AS base
MAINTAINER supernode.com
LABEL commitid=
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo 'Asia/Shanghai' >/etc/timezone
WORKDIR /app
EXPOSE 80
ENV LANG C.UTF-8
ENV ASPNETCORE_ENVIRONMENT DevTest

FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build
WORKDIR /src
copy . .

RUN dotnet restore "AI.BG.Manager.Project/AI.BG.Manager.Project.csproj"

WORKDIR "/src/AI.BG.Manager.Project"
RUN dotnet build "AI.BG.Manager.Project.csproj"  -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "AI.BG.Manager.Project.csproj" -c Release -o /app/publish

from base AS final
WORKDIR /app
COPY --from=publish /app/publish/ .

ENTRYPOINT ["dotnet", "AI.BG.Manager.Project.dll"]
EOF

        cat > ${WORKSPACE}/Dockerfile-gateway <<EOF
FROM mcr.microsoft.com/dotnet/aspnet:5.0-buster-slim AS base
MAINTAINER supernode.com
LABEL commitid=
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo 'Asia/Shanghai' >/etc/timezone
WORKDIR /app
EXPOSE 80
ENV LANG C.UTF-8
ENV ASPNETCORE_ENVIRONMENT DevTest

FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build
WORKDIR /src
copy . .

RUN dotnet restore "ApiGateway/AI.ApiGateway.csproj"

WORKDIR "/src/ApiGateway"
RUN dotnet build "AI.ApiGateway.csproj"  -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "AI.ApiGateway.csproj" -c Release -o /app/publish

from base AS final
WORKDIR /app
COPY --from=publish /app/publish/ .

ENTRYPOINT ["dotnet", "AI.ApiGateway.dll"]
EOF

        cat > ${WORKSPACE}/Dockerfile-swagger <<EOF
FROM mcr.microsoft.com/dotnet/aspnet:5.0-buster-slim AS base
MAINTAINER supernode.com
LABEL commitid=
RUN ln -sf /usr/share/zoneinfo/Asia/Shanghai /etc/localtime
RUN echo 'Asia/Shanghai' >/etc/timezone
WORKDIR /app
EXPOSE 80
ENV LANG C.UTF-8
ENV ASPNETCORE_ENVIRONMENT DevTest

FROM mcr.microsoft.com/dotnet/sdk:5.0 AS build
WORKDIR /src
copy . .

RUN dotnet restore "Swagger/AI.Swagger.csproj"

WORKDIR "/src/Swagger"
RUN dotnet build "AI.Swagger.csproj"  -c Release -o /app/build

FROM build AS publish
RUN dotnet publish "AI.Swagger.csproj" -c Release -o /app/publish

from base AS final
WORKDIR /app
COPY --from=publish /app/publish/ .

ENTRYPOINT ["dotnet", "AI.Swagger.dll"]
EOF

}


build_docker_images(){
    
    echo "*********************************************开始制作 ${service_name} 服务docker镜像**********************************************"
    docker -H tcp://192.168.1.124:4243 build -f ${dockerfile_path} -t ${image_name} .
    if [ $? = 0 ];then
        echo "docker 镜像构建成功"
    else
        echo "docker 镜像构建失败"
        exit 1
    fi 
    echo "*********************************************推送镜像：${image_name}**********************************************"
    docker -H tcp://192.168.1.124:4243 push ${image_name}
    echo "*********************************************${service_name} 服务docker镜像制作完成**********************************************"

}

deploy_service(){
    image_name=supernode:5000/${service_name}:${flag}
    swarm_service_name=aibg-devtest_${service_name}
    echo "*********************************************开始部署 ${service_name} 服务**********************************************"
    docker -H tcp://192.168.1.120:4243 service update --force --image ${image_name} ${swarm_service_name}
    echo "*********************************************部署 ${service_name} 完成**********************************************"
}

build_bg_manager(){
    cd ${WORKSPACE}
    image_name=supernode:5000/bg-manager:${flag}
    service_name=bg-manager
    dockerfile_path=${WORKSPACE}/Dockerfile-manager
    sed -i "s/LABEL commitid=/LABEL commitid=${comitid}/" ${dockerfile_path}
    build_docker_images
    deploy_service
}

build_bg_gateway(){
    
    image_name=supernode:5000/bg-gateway:${flag}
    service_name=bg-gateway
    dockerfile_path=${WORKSPACE}/Dockerfile-gateway
    sed -i "s/LABEL commitid=/LABEL commitid=${comitid}/" ${dockerfile_path}
    build_docker_images
    deploy_service
}

build_bg_swagger(){
    
    image_name=supernode:5000/bg-swagger:${flag}
    service_name=bg-swagger
    dockerfile_path=${WORKSPACE}/Dockerfile-swagger
    sed -i "s/LABEL commitid=/LABEL commitid=${comitid}/" ${dockerfile_path}
    build_docker_images
    deploy_service
}

delet_image(){
    docker -H tcp://192.168.1.124:4243 images|grep G|grep none|grep -v superset|awk '{print $3}'|xargs docker -H tcp://192.168.1.124:4243 rmi
    docker -H tcp://192.168.1.124:4243 system prune -af
}

run_main(){
    write_file
    build_bg_manager
    #build_bg_gateway
    #sleep 10
    #build_bg_swagger
    delet_image
}

run_main
