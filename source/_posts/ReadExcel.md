---
title: 上传/读取不规则数据的Excel
date: 2020-12-24 20:25:43
tags:
cover: https://s3.ax1x.com/2020/12/24/r2hMKP.jpg
copyright_author_href: https://github.com/KezhanW
copyright_url: https://kezhanw.github.io/2020/12/24/ReadExcel
---

# 上传文件
## 前台
定义一个文件选择器隐藏和一个导入按钮
```html
<a id="lr_import" class="btn btn-default"><i class="fa fa-sign-in"></i>&nbsp;导入</a>
<input id="filed" name="filed" type="file" style="display:none" accept=".xls,.xlsx">
```
重新定义一个导入按钮的目的在于便于添加额外逻辑  例如：选择不同的模板向后台传参
```js
 //导入
            $("#lr_import").click(function () {
                learun.layerForm({
                    id: 'form7',
                    title: '导入类型 ',
                    url: '/Fly_ZD/Fly_ZD_Head/ImportType',
                    width: 500,
                    height: 300,
                    callBack: function (id, index) {
                        var resData = top[id].acceptClick();
                        if (resData) {
                            ImportType = resData;
                            $("#filed").trigger("click");
                            return true;
                        } else {
                            learun.alert.error("请选择导入类型");
                        }
                    }
                });
            })
            $("#filed").on("change", function () {
                var fileM = document.querySelector("#filed");
                //获取文件对象，files是文件选取控件的属性，存储的是文件选取控件选取的文件对象，类型是一个数组
                var fileObj = fileM.files[0];
                //创建formdata对象，formData用来存储表单的数据，表单数据时以键值对形式存储的。
                var formData = new FormData();
                formData.append('filed', fileObj);
                learun.loading(true, '正在导入……');
                $.ajax({
                    url: "/Fly_ZD/Fly_ZD_Head/Import?ImportType=" + ImportType,
                    type: "post",
                    dataType: "json",
                    data: formData,
                    async: false,
                    cache: false,
                    contentType: false,
                    processData: false,
                    success: function (json_data) {
                        learun.loading(false);
                        $("#filed").val("")
                    },
                });
            });
```
## 后台
先将选择的文件上传到服务器 再读取该Excel
```
        /// <summary>
        /// 毕勤制单导入
        /// </summary>
        /// <param name="filed"></param>
        /// <returns></returns>
        public ActionResult Import(HttpPostedFileBase filed, string ImportType)
        {
            //服务器上的UpLoadFile文件夹必须有读写权限
            string target = Server.MapPath("/") + ($"/UploadFile/biqin/{DateTime.Now.Year}/{DateTime.Now.Month}/");//取得存储文件夹的路径
            string filename = filed.FileName;//取得文件名字
            string path = target + filename;//获取存储的目标地址
            //判断该文件路径是否存在没有则创建文件夹
            if (!Directory.Exists(target))
            {
                Directory.CreateDirectory(target);
            }
            //将文件保存到指定位置
            filed.SaveAs(path);
            //读取数据 存储数据
            GetHeadData(path, ImportType);
            return Success("");
        }
```
`GetHeadData()` 方法对读取到的数据存到数据库
```
			/// <summary>
			/// 毕勤制单获取数据
			/// </summary>
			/// <param name="filepath"></param>
			public void GetHeadData(string filepath, string ImportType)
			{
				try
				{
					DataSet gongsi = ReadExcelData(filepath);
					//定义两个datatable，并将传递来的dataset集中的两个表分别赋值给datatable
					//发票
					DataTable fp_data = gongsi.Tables[0];
					//箱单
					DataTable xd_data = gongsi.Tables[1];
					//主表信息
					Fly_ZD_ImportMainEntity HeadEntity = new Fly_ZD_ImportMainEntity();
					//发票商品信息
					List<Fly_ZD_ImportfapiaoEntity> fp_xx = new List<Fly_ZD_ImportfapiaoEntity>();
					//箱单商品信息
					List<Fly_ZD_ImportxiangdanEntity> xd_xx = new List<Fly_ZD_ImportxiangdanEntity>();
					if (ImportType == "VN")
					{
						HeadEntity = this.VNmain_xx(gongsi.Tables[0]);
						fp_xx = this.VNfp_xx(fp_data);
						xd_xx = this.VNxd_xx(xd_data);
					}
					else if (ImportType == "VNBI")
					{
						HeadEntity = this.VNBImain_xx(gongsi.Tables[0]);
						fp_xx = this.VNBIfp_xx(fp_data);
						xd_xx = this.VNBIxd_xx(xd_data);
					}
					else if (ImportType == "VA")
					{
						HeadEntity = this.VAmain_xx(gongsi.Tables[0]);
						fp_xx = this.VNfp_xx(fp_data);
						xd_xx = this.VNxd_xx(xd_data);
					}
					//保存到数据库中
					fly_ZD_HeadIBLL.SaveImport(HeadEntity, fp_xx, xd_xx);
				}
				catch (Exception e)
				{
	
				}
			}
```
`ReadExcelData()` 方法读取Excel 注意工作表要加$例如下文INV$，PK$
```
        /// <summary>
        /// 读取不规则的EXcel数据
        /// </summary>
        /// <param name="filepath"></param>
        /// <returns></returns>
        public static DataSet ReadExcelData(string filepath)
        {
            DataSet ds = new DataSet();//定义一个dataset用来存储查询到的数据
            string connStr = "Provider=Microsoft.Jet.OleDb.4.0;" + "Data Source=" + filepath + ";Extended Properties='Excel 8.0; HDR=YES; IMEX=1'";
            string sql_F = "SELECT * FROM [{0}]";
            System.Data.OleDb.OleDbConnection conn = null;
            System.Data.OleDb.OleDbDataAdapter xd_da = null;
            System.Data.OleDb.OleDbDataAdapter fp_da = null;
            System.Data.DataTable tblSchema = null;//实例化一个数据表用来存储表名
            System.Collections.Generic.IList<string> xd_tblNames = null;//定义一个List<string>变量 用来存储原始箱单表名
            System.Collections.Generic.IList<string> fp_tblNames = null;//定义一个List<string>变量 用来存储原始发票表名
            conn = new System.Data.OleDb.OleDbConnection(connStr);
            conn.Open();
            tblSchema = conn.GetOleDbSchemaTable(System.Data.OleDb.OleDbSchemaGuid.Tables, new object[] { null, null, null, "TABLE" });//获取所有表名
            xd_tblNames = new System.Collections.Generic.List<string>();
            fp_tblNames = new System.Collections.Generic.List<string>();
            string xd_tblname = "";
            string fp_tblname = "";
            for (int tblname = 0; tblname < tblSchema.Rows.Count; tblname++)
            {
                if (tblSchema.Rows[tblname][2].ToString() == "INV$")
                {
                    xd_tblname = tblSchema.Rows[tblname][2].ToString();//获取原始箱单表名
                }
                if (tblSchema.Rows[tblname][2].ToString() == "PK$")
                {
                    fp_tblname = tblSchema.Rows[tblname][2].ToString();//获取原始发票表名
                }
            }
            string[] xd_tblnames = { xd_tblname };//将原始箱单表名储存到数组里
            string[] fp_tblnames = { fp_tblname };//将原始发票表名储存到数组里
            xd_tblNames = new List<System.String>(xd_tblnames);//将原始箱单表名赋值给xd_tblNames
            fp_tblNames = new List<System.String>(fp_tblnames);//将原始发票表名赋值给xd_tblNames

            xd_da = new System.Data.OleDb.OleDbDataAdapter();
            fp_da = new System.Data.OleDb.OleDbDataAdapter();
            //循环获取全部的原始箱单数据
            foreach (string tblName in xd_tblNames)
            {
                xd_da.SelectCommand = new System.Data.OleDb.OleDbCommand(String.Format(sql_F, tblName), conn);
                try
                {
                    xd_da.Fill(ds, tblName);
                }
                catch
                {
                    if (conn.State == ConnectionState.Open)
                    {
                        conn.Close();
                    }
                }
            }
            //循环获取全部的原始发票数据
            foreach (string fp_tblName in fp_tblNames)
            {
                xd_da.SelectCommand = new System.Data.OleDb.OleDbCommand(String.Format(sql_F, fp_tblName), conn);
                try
                {
                    xd_da.Fill(ds, fp_tblName);
                }
                catch
                {
                    if (conn.State == ConnectionState.Open)
                    {
                        conn.Close();
                    }
                }
            }
            return ds;
        }
```