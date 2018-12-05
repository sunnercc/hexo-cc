---
title: POST
date: 2017-11-05 11:12:53
tags:
---

     #define Boundary @"--dagdsadgsahsdahdsfagd"
     
     Content-Type: multipart/form-data;boundary=--Boundary
     
     --Boundary\r\n
     Content-Disposition:form-data;name='参数key';filename='上传文件名'\r\n
     Content-Type:application/zip\r\n
     \r\n
     二进制文件数据\r\n
     
     --Boundary\r\n
     Content-Disposition:form-data;name='参数key'\r\n
     \r\n
     参数value\r\n
     
     --Boundary\r\n
     Content-Disposition:form-data;name='参数key'\r\n
     \r\n
     参数value\r\n
     
     --Boundary\r\n
     Content-Disposition:form-data;name='参数key'\r\n
     \r\n
     参数value\r\n
     
     --Boundary--\r\n
