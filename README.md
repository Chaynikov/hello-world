# hello-world
my first repo
		bool RefreshNumPos(OrderClass Order, EventArgs Args)
		{
//			if (Atechnology.ecad.Settings.idpeople == 4)
//				MessageBox.Show("addint1:"+Order.DocRow.addint1.ToString());
			
			//addint1 - Флаг - использование внешних номеров позиции, своими не перезатирается, только для импортируемых заказов!
			if(!Order.DocRow.Isaddint1Null()&&Order.DocRow.addint1==1)
				return false;
			
			string Err="1";
			try{
				//Order.LockTree();
				//Обновить номера конструкций
				int MaxConstrNum=0;
				foreach(ds_order.orderitemRow drOI in Order.ds.orderitem.Select())
				{
					if(!drOI.IsconstrnumNull()&&drOI.constrnum>MaxConstrNum)
						MaxConstrNum=drOI.constrnum;
				}
				Err="2";
				MaxConstrNum++;
				//Если конструкция составная, но объединяющей продукции нет, нумеруем одинаково
				foreach(ds_order.orderitemRow drOI in Order.ds.orderitem.Select("constrnum is null and parentid is null and idmodel is not null"))
				{
					ds_order.modelRow drM=Order.ds.model.FindByidmodel(drOI.idmodel);
					//Составная конструкция, родительской продукции еще нет, или связь еще не установлена
					if(!drM.IsidconstructionNull()){
						//Найти ранне установленный номер конструкции, добавить в новые модели
						int CurConstrNum=MaxConstrNum;
						foreach(ds_order.modelRow drM_All in Order.ds.model.Select("idconstruction="+drM.idconstruction)){
							if(!Order.ds.orderitem.FindByidorderitem(drM_All.idorderitem).IsconstrnumNull()){
								CurConstrNum=Order.ds.orderitem.FindByidorderitem(drM_All.idorderitem).constrnum;
							}
						}
						foreach(ds_order.modelRow drM_All in Order.ds.model.Select("idconstruction="+drM.idconstruction))
						{
							ds_order.orderitemRow dr=Order.ds.orderitem.FindByidorderitem(drM_All.idorderitem);
							if(dr.IsconstrnumNull()||dr.constrnum!=CurConstrNum)
								dr.constrnum=CurConstrNum;
						}
					}
				}
				Err="3";
				
				//Проставить номера конструкций для корневых узлов (только для составной конструкции и отдельных конструкций)
				foreach(ds_order.orderitemRow drOI in Order.ds.orderitem.Select("constrnum is null and parentid is null"))// and (idproductiontype=117 or productiontype_typ=1)"))
				{
					drOI.constrnum=MaxConstrNum++;
				}
				Err="4";
				//Проставить номера конструкций для подчиненных позиций
				foreach(ds_order.orderitemRow drOI in Order.ds.orderitem.Select("parentid is not null"))
				{
					Err="4.1 "+drOI.idorderitem;
					//Получить корневую позицию
					ds_order.orderitemRow RootRow=GetRootRow(Order,drOI);
					if(drOI.IsconstrnumNull()||drOI.constrnum!=RootRow.constrnum)
					{
						drOI.constrnum=RootRow.constrnum;
					}
					Err="4.2";
					ds_order.orderitemRow ParentRow=Order.ds.orderitem.FindByidorderitem(drOI.parentid);
					//Установить номера выделенных позиций, равных номеру родительской
					if(!drOI.IshideNull()&&drOI.hide&&!ParentRow.IsnumposNull())
					{
						Err="4.3";
						if(drOI.IsnumposNull()||drOI.numpos!=ParentRow.numpos)
						{
							//MessageBox.Show(""+drOI.numpos+";"+RootRow.numpos);
							drOI.numpos=ParentRow.numpos;
						}
					}
					Err="4.4";
				}
				Err="5";
				
				// Обновление номеров позиций (numpos)
				#region Обновление номеров позиций (numpos)
				
				
				// [NumPos] для позиций-конструкций совпадает с [ConstrNum]
				foreach ( ds_order.orderitemRow droi in Order.ds.orderitem.Select("idproductiontype = 107 " ) )
					if ( droi.IsnumposNull() || ( !droi.IsconstrnumNull() && droi.numpos != droi.constrnum ))
						droi.numpos = droi.constrnum;
				
				
				// [NumPos] для позиций-изделий идёт в порядке возрастания
				int numPos = 1;
				if(!Order.DocRow.Isaddint2Null())
					numPos=Order.DocRow.addint2;
				foreach ( ds_order.orderitemRow droi in Order.ds.orderitem.Select("(isnull(ispf,0) = 0 or parentid is null) and idmodel is not null", " constrnum asc, idorderitem asc " ) )
				{
					if ( droi.IsnumposNull() || droi.numpos != numPos )
						droi.numpos = numPos;
					numPos = numPos + 1;
				}
				Err="6";
				
				// [NumPos] для позиций-допов идёт в порядке возрастания
				foreach ( ds_order.orderitemRow droi in Order.ds.orderitem.Select("isnull(ispf,0) = 0 and idproductiontype <> 107 and idmodel is null ", " idorderitem asc " ) )
				{
					if ( droi.IsnumposNull() || droi.numpos != numPos )
						droi.numpos = numPos;
					numPos = numPos + 1;
				}
				Err="7";
				
				// [NumPos] для выделенных позиций - Из родительских идёт в порядке возрастания
				foreach ( ds_order.orderitemRow droi in Order.ds.orderitem.Select("isnull(ispf,0) = 1 ", " constrnum asc, idorderitem asc " ) )
				{
					if(droi.IsparentidNull()){
						if ( droi.IsnumposNull() || droi.numpos != numPos )
							numPos = numPos + 1;
					}
					else{
						int numpos=Order.ds.orderitem.FindByidorderitem(droi.parentid).numpos;
						if(droi.IsnumposNull()||numpos!=droi.numpos)
							droi.numpos = numpos;
					}
				}
				Err="8";

				// Корректировка номера позиции для спецификации
				foreach ( ds_order.orderitemRow droi in Order.ds.orderitem.Select())
					foreach ( ds_order.modelcalcRow drmc in Order.ds.modelcalc.Select(" idorderitem = " + droi.idorderitem.ToString()) )
						if ( drmc.Isorderitem_numposNull() || ( !droi.IsnumposNull() && droi.numpos != drmc.orderitem_numpos ))
							drmc.orderitem_numpos = droi.numpos;
				
				
				#endregion Обновление номеров позиций				
				Err="9";
			}
			catch(Exception e){
				//throw new Exception("Ошибка метода RefreshNumPos"+"\r\nОшибка:"+e.Message);
				MessageBox.Show("Ошибка метода RefreshNumPos"+"\r\nОшибка:"+e.Message+"\r\nТочка:"+Err);
			}
			finally{
				//Order.UnlockTree();
			}
			return false;
		}
