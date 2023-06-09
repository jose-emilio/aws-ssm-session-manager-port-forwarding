# **<em>Local Port-Forwarding</em> con AWS Systems Manager Session Manager**

## **Objetivo**

La **redirección de puertos**, también denominada <em>**Port-Forwarding**</em> es una técnica que permite redirigir el tráfico de red de un puerto de un nodo a otro puerto de otro nodo de una red. Con el protocolo SSH es posible realizar una redirección de puertos, siendo este mecanismo utilizado con frecuencia para evadir restricciones de cortafuegos y otros filtros de red. SSH permite varios tipos de redirecciones:

* **Redirección local de puertos** (<em>Local Port-Forwarding</em>)
* **Redirección remota de puertos** (<em>Remote Port-Forwarding</em>)
* **Redirección dinámica de puertos** (Proxy SOCKS)

En este repositorio se demuestra el funcionamiento de un túnel SSH con <em>Local Port Forwarding</em> mediante AWS SSM Session Manager, de forma segura, sin necesidad de exponer el puerto SSH de la máquina que creará el túnel. Para ello, se crearán dos instancias EC2 en subredes privadas, tras un NAT Gateway. Una de las instancias (instancia A), se utilizará para crear un túnel SSH a un servicio HTTP alojado en la segunda instancia (instancia B). Por último se comprobará el tráfico desde una máquina externa hacia el servidor HTTP a través del túnel.

## **Requerimientos**

* Una cuenta de AWS o un sandbox de <em>AWS Academy Learner Lab</em> 
* Un entorno con sistema operativo Linux y con acceso programático a AWS configurado

## **Arquitectura propuesta**

<p align="center">
  <img src="images/port-forwarding.png">
</p>

## **Servicios utilizados**

* **Amazon VPC** para la creación de la infraestructura de red
* **Amazon EC2** para el despliegue de las instancias A y B
* **AWS Systems Manager** para registrar las instancias EC2 y abrir un túnel seguro con redirección de puertos 

## **Instrucciones**

1. Se establece la región donde se creará la infraestructura (cambiar el valor de la variable de entorno por el apropiado):
        
        REGION=us-east-1

2. Se crea un bucket de Amazon S3 donde se almacenarán los archivos necesarios para el despliegue (cambiar el nombre del bucket por el apropiado). AWS CloudFormation utilizará dicho bucket.

        aws s3 mb s3://cloudformation-us-east-2-jevs --region $REGION
        BUCKET=cloudformation-us-east-2-jevs

3. Empaquetar la plantilla de AWS CloudFormation:

**Nota importante**: Para deplegar la plantilla en un entorno de **AWS Academy Learner Lab**, debe indicarse en el parámetro `--template-file` la plantilla `port-forwarding-learner-lab.yaml` en lugar de `port-forwarding.yaml`.

        aws cloudformation package --template-file port-forwarding.yaml --s3-bucket $BUCKET --output-template-file port-forwarding-transformed.yaml --region $REGION

4. Desplegar la plantilla de AWS CloudFormation:

        aws cloudformation deploy --template-file port-forwarding-transformed.yaml --stack-name port-forwarding-stack --capabilities CAPABILITY_IAM --region $REGION

5. Se obtiene el ID de la instancia A:

        id1=$(aws cloudformation describe-stacks --stack-name port-forwarding-stack --query 'Stacks[0].Outputs[?OutputKey==`Id1`].OutputValue' --output text --region $REGION)

6. Se obtiene la IP de la instancia B:

        ip2=$(aws cloudformation describe-stacks --stack-name port-forwarding-stack --query 'Stacks[0].Outputs[?OutputKey==`Ip2`].OutputValue' --output text --region $REGION)

7. Tras unos minutos, la pila de recursos se habrá creado. Ahora sólo será necesario crear una sesión contra la instancia A desde AWS SSM Session Manager que actuará como túnel SSH contra el puerto 80 de la instancia B. Para ello es necesario que en la máquina local se haya instalado previamente el plugin de AWS SSM Session Manager (https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with-install-plugin.html). El túnel SSH mapeará el puerto 10000 local al puerto 80 de la instancia B a través de la instancia A. Para ello, se indicará a AWS SSM Session Manager que al iniciar la sesión se ejecute un automatismo que cree el túnel, que se parametrizará con los valores anteriores:

        aws ssm start-session --target $id1 --document-name AWS-StartPortForwardingSessionToRemoteHost --parameters "portNumber"=["80"],"localPortNumber"=["10000"],"host"=[$ip2] --region $REGION

8. Por último, sólo resta abrir un navegador o cliente HTTP y acceder a la URL:

        curl http://localhost:10000
