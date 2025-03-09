
//attribut add component
export class AttributeAddComponent implements OnInit {
  myControl = new UntypedFormControl();
  errorControl = new UntypedFormControl('', Validators.required);
  selected = '';
  editable: boolean = false;
  submited: boolean = false;
  id:any;
  selectedCar:any;
  selectedars = [];
  selectedCars = [3,5];
  templatetypes: any[] = []
  categoryTemplateForm : FormGroup = this.formBuilder.group({
      MRHRC_TMP_NM:['',[Validators.required,Validators.maxLength(255)]],
      ID_MRHRC_TMP_TYP:[null,[Validators.required]],
      MRHRC_TMP_STATUS:['A',[]],
      custom_field: this.formBuilder.array([])
  })
  fields: any[] = []
  cars = [  ];
  baseUrl: string = environment.BASE_URL;
  lang: string = '';
  backClicked() {
    this._location.back();
  }
  constructor(public dialog: MatDialog,private cateTemplateCreateCommonService: CommonService,private _location: Location,private router:Router,private formBuilder: FormBuilder,private apiService: ApiService,private activatedRoute: ActivatedRoute) { }
  ngOnInit(){
    this.lang = this.cateTemplateCreateCommonService.getItem('defaultLang') ?? '';
    this.cateTemplateCreateCommonService.setGlobalRouterLaoder(true)
    this.getTemplateTypes()
    this.editable = this.activatedRoute.snapshot.url.toString().includes('edit')
    this.id = this.activatedRoute.snapshot.params['id'];
    if(this.editable){
      this.getCategoryTemplate()
    }
    
  }
  get custom_field():FormArray{
    return this.categoryTemplateForm.get('custom_field') as FormArray;
  }
  get attControls() {
    return this.categoryTemplateForm.controls;
  }
  getCategoryTemplate(){
    this.apiService
        .get(
          `${MASTER_URL.ATTRIBUTES_URL}?ID_MRHRC_TMP=${this.id}`
        )
        .subscribe({
          next: (categorytemplateResponse: any) => {
            console.log("template",categorytemplateResponse);
            this.categoryTemplateForm.patchValue({...categorytemplateResponse.data[0],ID_MRHRC_TMP:this.id});
            const customFields=categorytemplateResponse.data[0].custom_field.filter((field:any)=>field?.BA_CFF_NM && field?.BA_CFF_NM!=undefined && field?.BA_CFF_NM !=null);
            if(customFields?.length > 0){
              for(let attribute of categorytemplateResponse.data[0].custom_field){
                let dval = null
                let fmval: string[] = []
                if(attribute.CFF_TYP_NM=='Dropdown'){
                  dval =this.addDropDownToTemp(attribute)
                }
                if(attribute.CFF_TYP_NM=='Dropdown Multiselect'){
                  fmval =this.addDropMultiToTemp(attribute)
                }
                this.addCustomField({...attribute,
                  D_CFF_VAL: dval,
                  DM_CFF_VAL: new FormControl([...fmval])
                },true)
              }
            }
            
            this.cateTemplateCreateCommonService.setGlobalRouterLaoder(false)
            console.log("whole form",this.categoryTemplateForm.value);

          },
          error: (categorytemplateError: any) => {
            this.cateTemplateCreateCommonService.setGlobalRouterLaoder(false)
          },
        });
  }
  addDropDownToTemp(temp?:any){
    for(let value of temp.custom_value){
      if(value.isdefault){
        return value?.MRHRC_TMP_CNT_VL ? value?.MRHRC_TMP_CNT_VL :value?.CFF_VAL;
      }
    }
  }
  addDropMultiToTemp(temp?:any){
    let arr:any[] = []
    for(let value of temp.custom_value){
      if(value.isdefault){
        arr.push(value?.MRHRC_TMP_CNT_VL ? value?.MRHRC_TMP_CNT_VL :value?.CFF_VAL)
      }
    }
    return arr
  }
  addCategoryTemplate(){
    this.categoryTemplateForm.markAllAsTouched();
    this.submited = true
    console.log("edit",{...this.categoryTemplateForm.value,ID_MRHRC_TMP:this.id});
    if(this.categoryTemplateForm.valid){ 
      const url=`${MASTER_URL.ATTRIBUTES_URL}${this.editable ? "edit/" + this.id : "add"}/`;
      const payload=this.editable?{...this.categoryTemplateForm.value,ID_MRHRC_TMP:this.id}:
      this.categoryTemplateForm.value;
      if(!this.editable){
        this.apiService.post(url,payload).subscribe({
          next:res=>{
            this.cateTemplateCreateCommonService.openSnackbar(res?.msg)
            this.router.navigate([this.lang+'/pages/admin/master-setting/attribute/list'])
          }
        });
      } else {
        this.apiService.put(url,payload).subscribe({
          next:res=>{
            this.cateTemplateCreateCommonService.openSnackbar(res?.msg)
            this.router.navigate([this.lang+'/pages/admin/master-setting/attribute/list'])
          }
        });
      }
      }
  }
  addCustomField(data:any,isEdit?:boolean){
    let cvalues : any[] = data?.customformfield_values ? data?.customformfield_values : data?.custom_value;
    this.custom_field.push(this.formBuilder.group({...data,
      custom_value: this.formBuilder.array(isEdit ? [...this.modifyObject(data?.custom_value)]:[...cvalues]),
    }));
        
  }

  modifyObject(data:any[]):any[]{
    data.forEach(x=>{
      x.CFF_VAL=x?.MRHRC_TMP_CNT_VL
    });
    return data;
  }

  
  getLabel(value:any){
    return value
  }
  editCustomField(data: any,index:number){
    const dialogRef = this.dialog.open(CategoryCustomfieldDialogComponent, {
      width:'100%',
      maxWidth:'600px',
      height:'400px',
      panelClass: ['pop-up','sm-pop'],
      data:{
        selectedData: data.value,
      },
      disableClose: true
    });    
    dialogRef.afterClosed().subscribe(result => {
      if(result.status=="edited"){
        this.custom_field.removeAt(index)
        this.custom_field.insert(index,this.formBuilder.group({
          ...result,
          custom_value: this.formBuilder.array([...(result?.customformfield_values ?? [])])

        }))  
      }
      if(result.status=="editnew"){
        this.custom_field.removeAt(index)
        this.custom_field.insert(index,this.formBuilder.group({
          ...result,
          custom_value: this.formBuilder.array([...(result?.customformfield_values ?? [])])

        }))
        this.customFieldsDialog(true)
      }
    });
  }

  
  removeCustomField(index:number){
    this.custom_field.removeAt(index)
  }
  getTemplateTypes(){
    this.apiService.get(`${MASTER_URL.ATTRIBUTES_TYPE_DROPDOWN_URL}`)
    .subscribe({
      next: (categorytemplateResponse: any) => {
        console.log('fields',categorytemplateResponse);
        this.templatetypes = categorytemplateResponse.data
        this.cateTemplateCreateCommonService.setGlobalRouterLaoder(false)
      },
      error: (categorytemplateError: any) => {
        this.cateTemplateCreateCommonService.setGlobalRouterLaoder(false)
      },
    });
  }
  customFieldsDialog(open?:boolean) {
    const dialogRef = this.dialog.open(CategoryCustomfieldDialogComponent, {
      width:'100%',
      maxWidth:'600px',
      panelClass: ['pop-up','sm-pop'],
      data:{
        isOpen: open? true: false
      },
      disableClose: true
    });

    dialogRef.afterClosed().subscribe(result => {
      if(result.status=="added"){
        console.log("adddata",result);
        
        this.addCustomField(result)
      }
 
      if(result.status=="addnew"){
        this.addCustomField(result)

        this.customFieldsDialog(true)
      }
    });

  }
  getSelectedFields(index:number){
    console.log("sfields",this.custom_field.at(index).value.custom_value.filter((cv:any)=> cv.isdefault));
    
    return this.custom_field.at(index).value.custom_value.filter((cv:any)=> cv.isdefault)
  }
  checkField(field:any){
    let value = field?.custom_value[0]?.MRHRC_TMP_CNT_VL ? field?.custom_value[0]?.MRHRC_TMP_CNT_VL : field?.custom_value[0]?.CFF_VAL;
    if(field?.CFF_TYP_NM=='Date Picker' && value){
      const date = new Date(value),
        mnth = ("0" + (date.getMonth() + 1)).slice(-2),
        day = ("0" + date.getDate()).slice(-2);

      return [date.getFullYear(), mnth, day].join("-");
      

    }
    return value
  }
  isSticky: boolean = false;
  pSp = window.pageYOffset;
  cSp=window.pageYOffset;
  @HostListener('window:scroll', ['$event'])
  checkScroll() {
   this.cSp = window.pageYOffset;
   if(this.cSp > 20){
      this.isSticky = this.cSp < this.pSp;
    }else{
     this.isSticky = false
    }
    this.pSp = this.cSp; 
  }
}

//======================================================

//ad attribute template

<div class="common-page-body add-attributes-page common-gap">
	<form [formGroup]="categoryTemplateForm" class="add-attributes-inner">

		<div class="form-box ">
			<div class="form-box-head">
				<h5>{{'masterSetting.attributes.addAttributeGroup' | translate}}</h5>
			</div>
			<div class="form-box-body">

				<div class="grid-row">
					<div class="grid-column mobile-grid-column">
						<mat-form-field appearance="outline">
							<mat-label>{{'masterSetting.attributes.templateName' | translate}}</mat-label>
							<input matInput formControlName="MRHRC_TMP_NM" required="">
							
						</mat-form-field>
						<mat-error text-align="right" align="end" class="text-danger"
							*ngIf="submited && attControls['MRHRC_TMP_NM']?.hasError('required') && attControls['MRHRC_TMP_NM']?.touched">
							This field is required
						</mat-error>
						<mat-error text-align="right" align="end" class="text-danger"
							*ngIf="(submited && attControls['MRHRC_TMP_NM'].errors?.['maxlength']) || (attControls['MRHRC_TMP_NM'].errors?.['maxlength'] && (attControls['MRHRC_TMP_NM'].dirty || attControls['MRHRC_TMP_NM'].touched))">
							This field should be maximum 255 character long.
						</mat-error>
					</div>
					<div class="grid-column mobile-grid-column">
						<mat-form-field class="field-drop" appearance="outline">
							<mat-label>{{'masterSetting.attributes.tempType' | translate}}</mat-label>
							<mat-select formControlName="ID_MRHRC_TMP_TYP">
								<mat-option *ngFor="let type of templatetypes"
									[value]="type.ID_MRHRC_TMP_TYP">{{type.MRHRC_TMP_TYP_NM}}</mat-option>
									<mat-option *ngIf="templatetypes?.length === 0" class="select-drop-option">{{'common.noDataFound' | translate}}</mat-option>
							</mat-select>
						</mat-form-field>
						<mat-error text-align="right" align="end" class="text-danger"
							*ngIf="submited && attControls['ID_MRHRC_TMP_TYP']?.hasError('required') && attControls['ID_MRHRC_TMP_TYP']?.touched">
							This field is required
						</mat-error>
					</div>
					<div class="grid-column mobile-grid-column">
						<mat-form-field class="field-drop" appearance="outline">
							<mat-label>{{'common.status' | translate}}</mat-label>
							<mat-select formControlName="MRHRC_TMP_STATUS">
								<mat-option [value]="'A'">{{'common.active' | translate}}</mat-option>
								<mat-option [value]="'I'">{{'common.inactive' | translate}}</mat-option>
							</mat-select>
						</mat-form-field>
					</div>
				</div>

			</div>
		</div>
		<div class="form-box common-gap">
			<div class="form-box-head">
				<h5>{{'masterSetting.attributes.attributes' | translate}}</h5>
				<button class="outline-btn" type="submit" (click)="customFieldsDialog()">
					<mat-icon>add</mat-icon>{{'common.add' | translate}}
					{{'masterSetting.attributes.attributes' | translate}}
				</button>
			</div>

			<div class="form-box-body">

				<div class="grid-row">
					<div class="grid-column mobile-grid-column" formArrayName="custom_field"
						*ngFor="let field of custom_field?.controls;let i=index">
						<div class="field-outer" [formGroupName]="i">
							<div class="field_grp_area">
								<mat-form-field
									*ngIf="field.value?.CFF_TYP_NM!='Dropdown' && field.value?.CFF_TYP_NM!='Dropdown Multiselect'"
									appearance="outline" readonly='true'>
									<mat-label
										*ngIf="field.value?.BA_CFF_LBL">{{custom_field.at(i).value.BA_CFF_LBL}}</mat-label>
									<input matInput [value]="checkField(field.value)" readonly>
								</mat-form-field>

								<mat-form-field class="field-drop" *ngIf="field.value?.CFF_TYP_NM=='Dropdown'" appearance="outline"
									readonly='true'>
									<mat-label
										*ngIf="field.value?.BA_CFF_LBL">{{custom_field.at(i).value.BA_CFF_LBL}}</mat-label>
									<mat-select formControlName="D_CFF_VAL">
										<mat-option *ngFor="let fitem of field.value.custom_value"
											[value]="fitem?.MRHRC_TMP_CNT_VL ? fitem?.MRHRC_TMP_CNT_VL :fitem?.CFF_VAL">{{fitem?.MRHRC_TMP_CNT_VL ? fitem?.MRHRC_TMP_CNT_VL :fitem?.CFF_VAL}}</mat-option>
									</mat-select>
								</mat-form-field>

								<mat-form-field class="field-drop multiple_option" *ngIf="field.value?.CFF_TYP_NM=='Dropdown Multiselect'"
									appearance="outline" readonly='true'>
									<mat-label
										*ngIf="field.value?.BA_CFF_LBL">{{custom_field.at(i).value.BA_CFF_LBL}}</mat-label>
									<mat-select formControlName="DM_CFF_VAL" [multiple]="true">
										<mat-option *ngFor="let fitem of field.value.custom_value"
											[value]="fitem?.MRHRC_TMP_CNT_VL ? fitem?.MRHRC_TMP_CNT_VL :fitem?.CFF_VAL">{{ fitem?.MRHRC_TMP_CNT_VL ? fitem?.MRHRC_TMP_CNT_VL :fitem?.CFF_VAL }}
											<mat-checkbox *ngIf="false"
												[checked]="fitem.isdefault">{{ fitem?.MRHRC_TMP_CNT_VL ? fitem?.MRHRC_TMP_CNT_VL :fitem?.CFF_VAL }}</mat-checkbox>
										</mat-option>
									</mat-select>
								</mat-form-field>
							</div>

							<div class="edit-dlt">
								<button (click)="editCustomField(field,i)" matTooltip="Basic">
									<mat-icon class="material-icons-outlined">mode_edit</mat-icon>
								</button>
								<button (click)="removeCustomField(i)" matTooltip="Basic">
									<mat-icon>delete</mat-icon>
								</button>
							</div>
						</div>
					</div>
					<div class="col-sm-6 col-xl-3" style="font-size: 13px;margin-bottom: 10px;"
						*ngIf="!custom_field?.controls?.length">
						{{'masterSetting.attributes.noAttributes' | translate}}
					</div>
				</div>
			</div>


		</div>
	</form>
	<div class="next-btn-area">
		<div></div>
		<div>
			<button class="cancel_btn" appPreviousMenu (click)="backClicked()">{{'common.cancel' | translate}}</button>
		<button class="primary-btn next_btn"
			(click)="addCategoryTemplate()">{{editable?('common.edit' | translate): ('common.add' | translate)}}</button>
		</div>
	</div>
</div>


----attribute add component css-------------------

@import "../../../../../../../host/src/assets/styles/variable.scss";

.add-attributes-page {
  @include page-height-one;

  .add-attributes-inner {
    @include page-sub-height-one;
    overflow-y: auto;

    @media (max-width: 1199px) {
      padding: 0 10px 65px 10px;
    }
  }

  .grid-row {
    display: flex;
    flex-wrap: wrap;
    margin: 0px -7.5px 0 -7.5px;

    .grid-column {
      flex-basis: 33.33%;
      max-width: 33.33%;
      padding: 0 7.5px;
      box-sizing: border-box;

      @media (max-width: 991px) {
        flex-basis: 50%;
        max-width: 50%;
      }

      &.mobile-grid-column {
        @media (max-width: 575px) {
          flex-basis: 100%;
          max-width: 100%;
        }
      }
    }
  }

  .field-outer {
    border: 1px dashed #c9c9c9;
    border-radius: 3px;
    background: $sky-three;
    padding: 12px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    margin-bottom: 15px;
    .field_grp_area{
      width: 85%;
      @media(max-width:1440px){
        width: 80%;
      }
    }
    .mat-mdc-form-field {
      margin-bottom: 0;
    
    }
    button{
        padding: 0;
        margin-left: 10px;
        &:hover{
          .mat-icon{
              color: $primary-color;
          }
      }
        .mat-icon{
            color: $secondary-color;
            font-size: 18px;
            height: 18px;
            width: 18px;
        }
    }
    .edit-dlt{
        width: calc(100% - 85%);
        display: flex;
        justify-content: end;
        @media(max-width:1440px){
            width: calc(100% - 80%);
          }
    }
  }
}
.multiple_option{
  ::ng-deep{
  .mat-pseudo-checkbox-checked{
    background-color: $primary-color !important;
  }
}
}


//===============================

export class CategoryCustomfieldDialogComponent implements OnInit {
  @Input() fieldFormData: any = null;
  selectedName: string = '';
  selectedType: string = '';
  fieldTypes: any[] = []
  fields: any[] = [];
  newfields: any[] = []
  stypes: string[] = []
  isAnother: boolean = false
  currentField?: string;
  submitted:boolean=false;
  dataSubmitted:boolean=false;
  categoryCustomFieldForm: FormGroup = new FormGroup({});

  constructor(@Inject(MAT_DIALOG_DATA) public data: any, private catCommonService: CommonService, private dialogRef: MatDialogRef<CategoryCustomfieldDialogComponent>, private formBuilder: FormBuilder, private apiService: ApiService) { }
  ngOnInit() {
    this.makeForm();
    this.initialDataLoader();
    console.log("dialog data", this.data?.selectedData);
    if (this.data?.selectedData) {
      delete this.data?.selectedData['customformfield_values']
      console.log("dialog data2", this.data?.selectedData);
      this.categoryCustomFieldForm.patchValue(this.data?.selectedData)
      this.selectedType = this.data?.selectedData.CFF_TYP_NM
      for (let field of this.data.selectedData.custom_value) {

        this.addFieldValue(field.CFF_VAL, field.isdefault, field.ID_BA_CFF_VAL)
        this.updateCheckData(field.CFF_VAL, field.isdefault)

      }
      this.removeDefaultValueValidators(this.selectedType);
    }
    else {
      this.addFieldValue();

    }

  }

  makeForm() {
    this.categoryCustomFieldForm = this.formBuilder.group({
      BA_CFF_NM: [null, Validators.required],
      ID_BA_CFF_TYP: null,
      ID_BA_CFF: null,
      ID_MRHRC_TMP_CNT: null,
      CFF_TYP_NM: [null, Validators.required],
      D_CFF_VAL: '',
      DM_CFF_VAL: '',
      customformfield_values: this.formBuilder.array([]),
      BA_CFF_LBL: '',
      BA_CFF_DE: '',
      variataion_field: false,
      isfilterable: false,
      ismandatory:false
    });
  }

  initialDataLoader() {
    if (this.data.isOpen)
      this.isAnother = this.data?.isOpen
    this.getFieldTypes()
    this.getFormFields()

  }
  getFieldTypes() {

    this.apiService.get(`${MASTER_URL.CUSTOM_FORM_FIELD_TYPE_URL}?ordering=CFF_TYP_NM`).subscribe({
      next: (res: any) => {
        if (res) {
          this.fieldTypes = res.data
        }
      },
      error: (err: any) => { },
    });
  }
  get DM_CFF_VAL(): FormArray {
    return this.categoryCustomFieldForm.get('DM_CFF_VAL') as FormArray;
  }
  get customformfield_values(): FormArray {
    return this.categoryCustomFieldForm.get('customformfield_values') as FormArray;
  }

  getFormFields() {
    this.apiService.get(`${MASTER_URL.CUSTOM_FORM_FIELD_URL}?ordering=BA_CFF_NM&offset=0&page_size=1000`)
      .subscribe({
        next: (res: any) => {
          if (res) {
            console.log("field data", res);

            this.fields = res.data
          }
        },
        error: (err: any) => {
          console.log(err);
        },
      });
  }
  setVariation() {
    console.log("changefilter");

    
  }
  searchFields(event: any) {

    let value: string = event?.$ngOptionLabel ? event.$ngOptionLabel : event;
    console.log('value',value)
    this.categoryCustomFieldForm.patchValue({
      BA_CFF_NM: value,
      BA_CFF_LBL:value
    })


    if (!this.newfields.includes(value)) {
      let field = this.fields.filter((item: any) => item?.ID_BA_CFF == value)[0]
      this.categoryCustomFieldForm.patchValue({ ...field, ID_BA_CFF_TYP: field?.ID_BA_CFF_TYP })
      this.customformfield_values.clear()
      this.selectedType = field?.CFF_TYP_NM;
     
      for (let item of (field?.CustomFormFieldValue ?? [])) {
        this.addFieldValue(item?.CFF_VAL, item?.isdefault, item.ID_BA_CFF_VAL)
        this.updateCheckData(item.CFF_VAL, item.isdefault)
      }
      this.removeDefaultValueValidators(this.selectedType);
    }

  }

  addNewTag(tag: any) {
    this.fields = [{ BA_CFF_NM: tag, ID_BA_CFF: null }, ...this.fields]
    this.newfields.push(tag)
    console.log("tag", tag, this.fields);

    // this.fields.push({BA_CFF_NM:tag,ID_BA_CFF:null})

  }
  updateCheckData(text?: any, check?: boolean, index?: number) {
    let val = text
    if (text?.value?.CFF_VAL) {
      val = text.value.CFF_VAL;
    }
    console.log("value", val);

    if (this.selectedType == 'Dropdown' && check) {

      this.categoryCustomFieldForm.patchValue({
        D_CFF_VAL: val
      })
    }
    if (this.selectedType == 'Dropdown Multiselect' && check) {
      this.stypes.push(val)
      this.categoryCustomFieldForm.patchValue({
        DM_CFF_VAL: new FormControl([...this.stypes])
      })
      // push(this.formBuilder.group(text))
    }
  }
  addFieldValue(text?: string, check?: boolean, id?: string) {

    this.customformfield_values.push(this.formBuilder.group({
      CFF_VAL: [text ?? '', [Validators.required]],
      ID_BA_CFF_VAL: id ?? null,
      isdefault: check ?? false,
    }))
  }
  checkDefault(field: any) {
    if (this.selectedType == 'Dropdown')
      return this.categoryCustomFieldForm.value.D_CFF_VAL == field.value.CFF_VAL
    else if (this.selectedType == 'Dropdown Multiselect') {
      return this.stypes.includes(field.value.CFF_VAL)
    }
    else return true
  }
  removeFieldValue(index: number) {
    this.customformfield_values.removeAt(index)
  }
  fieldTypeOnchange(event: any) {
    let ffilter = this.fieldTypes.filter((item: any) => item.CFF_TYP_NM == event)[0];
    this.selectedType = event;
    this.categoryCustomFieldForm.patchValue({
      ID_BA_CFF_TYP: ffilter?.ID_BA_CFF_TYP,
      variataion_field: false,
      isfilterable: false,
      ismandatory:false
    });
    this.customformfield_values.clear();
    this.addFieldValue();
    if(this.customformfield_values.controls?.length){
      this.removeDefaultValueValidators(event);
    }


  }

  removeDefaultValueValidators(customeFieldType:string) {
    if((customeFieldType == 'Text Box' || customeFieldType == 'Text Area' || customeFieldType == 'Date Picker')){
      console.log('his.customformfield_values',this.customformfield_values)
      this.customformfield_values.controls[0].get('CFF_VAL')?.removeValidators([Validators.required]);
    }
  }


  addCategoryFunction(isAddNew?: boolean) {
    this.submitted=true;
    if((this.categoryCustomFieldForm.value.BA_CFF_NM == 'Color' || this.categoryCustomFieldForm.value.BA_CFF_NM == 'Size')) 
        this.customformfield_values.clear();

    console.log(this.categoryCustomFieldForm)
    this.categoryCustomFieldForm.markAllAsTouched();
    if (this.categoryCustomFieldForm.valid) {
      this.dataSubmitted=true;
      if (this.newfields.includes(this.categoryCustomFieldForm.value.BA_CFF_NM)) {
        this.apiService.post(`${MASTER_URL.CUSTOM_FORM_FIELD_URL}add/`, { ...this.categoryCustomFieldForm.value, DM_CFF_VAL: this.DM_CFF_VAL?.value?.value ,BA_CFF_LBL:this.categoryCustomFieldForm.value['BA_CFF_NM']})
          .subscribe({
            next: (res: any) => {
              this.submitted=false;
              this.dataSubmitted=false;
              if (res)
                this.customFieldAddLogic(res, isAddNew)
            },
            error: (err: any) => {
              this.submitted=false;
              this.dataSubmitted=false;
            },
          });
      } else {
        this.addNewCustomFieldLogic(isAddNew)
      }
    }
  }
  customFieldAddLogic(res: any, isAddNew?: any) {
    let message = res?.msg || `Category Attribute Created successfully`;
    this.catCommonService.openSnackbar(message);
    this.customformfield_values.clear()
    for (let value of res.data.customformfield_values) {
      this.addFieldValue(value.CFF_VAL, value.isdefault, value.ID_BA_CFF_VAL)
    }
    this.categoryCustomFieldForm.patchValue({
      ID_BA_CFF: res.data.ID_BA_CFF
    })
    this.addNewCustomFieldLogic(isAddNew)
  }
  addNewCustomFieldLogic(isAddNew: any) {
    if (this.data?.selectedData) {
      if (isAddNew)
        this.dialogRef.close({ status: "editnew", ...this.categoryCustomFieldForm.value })
      else
        this.dialogRef.close({ status: "edited", ...this.categoryCustomFieldForm.value });
    }
    else if (!this.data?.selectedData) {
      if (isAddNew)
        this.dialogRef.close({ status: "addnew", ...this.categoryCustomFieldForm.value })
      else
        this.dialogRef.close({ status: "added", ...this.categoryCustomFieldForm.value })
    }
  }
  disableOption(type: string) {
    if (type == this.selectedType) {
      if (this.categoryCustomFieldForm.value.BA_CFF_NM == 'Color' || this.categoryCustomFieldForm.value.BA_CFF_NM == 'Size') return false
      return true
    }
    else return false
  }
  checkSelectedFieldType(type: string) {

    if (type == this.selectedType) {
      return true
    }
    else return false
  }
  closeDialog() {
    this.dialogRef.close({ status: "cancelled" });
  }

  disableCheckbox() {
    const data: string[] = ['Text Box', 'Date Picker', 'Text Area'];
    return data.includes(this.categoryCustomFieldForm.get('CFF_TYP_NM')?.value);
  }

  customFieldSetDefaultValue(selectedFieldType:string,index:number) {
    console.log('selectedFieldType',selectedFieldType)
    console.log(index)
    console.log(this.categoryCustomFieldForm)
    if(this.customformfield_values.controls[0].get('CFF_VAL')?.value && this.categoryCustomFieldForm.get('CFF_TYP_NM')?.value==selectedFieldType){
      this.customformfield_values.controls[0].patchValue({
        isdefault:true
      })
    }
  }
}


//========================

<div class="add-color-modal">
  <div class="form-box">
    <div class="form-box-head">
      <h5>{{ data?.selectedData ? ('common.edit' | translate) : ('common.add' | translate) }} {{'masterSetting.attributes.attributes'| translate}}</h5>
      <button (click)="closeDialog()" class="close_btn" mat-dialog-close>
        <mat-icon>close</mat-icon>
      </button>
    </div>
    <div class="form-box-body modal-body">
      <form [formGroup]="categoryCustomFieldForm">
        <div class="field-drop">
          <ng-select formControlName="BA_CFF_NM" addTagText="Add New Field Name" [addTag]="addNewTag"
          (change)="searchFields($event)" appearance="outline" placeholder="Custom Field Name" [closeOnSelect]="true"
          [hideSelected]="true" appendTo="body" padding="no" required="">
          <ng-option *ngFor="let field of fields" [value]="field.ID_BA_CFF">{{
            field.BA_CFF_NM
          }}</ng-option>
          <ng-template ng-tag-tmp let-search="searchTerm">
            <div (click)="addNewTag(search)">
              <strong [ngStyle]="{ color: '#0000FF' }" #custom>{{'masterSetting.attributes.AddNewFieldName'| translate}}</strong>:
              <span>{{ search }} </span>
            </div>
          </ng-template>
        </ng-select>
        <mat-error *ngIf="submitted && categoryCustomFieldForm.get('BA_CFF_NM')?.hasError('required') && categoryCustomFieldForm.get('BA_CFF_NM')?.touched"   style="padding-top: 8px;" class="text-danger" text-align="right" align="end">
          {{'globalSetting.autoResponder.TemplateNameRequired'| translate}}
        </mat-error>
        </div>

        <ng-select class="common-gap" formControlName="CFF_TYP_NM" appearance="outline" placeholder="Field Type"
          (change)="fieldTypeOnchange($event)" [ngClass]="{
            'disabled-checkbox': !newfields.includes(
              categoryCustomFieldForm.value.BA_CFF_NM
            )
          }" [closeOnSelect]="true" [hideSelected]="true" appendTo="body" padding="no">
          <ng-option *ngFor="let field of fieldTypes" [value]="field.CFF_TYP_NM">{{ field.CFF_TYP_NM }}</ng-option>
        </ng-select>
        <mat-error *ngIf="submitted && categoryCustomFieldForm.get('CFF_TYP_NM')?.hasError('required') && categoryCustomFieldForm.get('CFF_TYP_NM')?.touched"   style="padding-top: 8px;" class="text-danger" text-align="right" align="end">
          {{'masterSetting.attributes.fieldTypeRequired'| translate}}
        </mat-error>

        <mat-form-field class="common-gap" appearance="outline" color="primary">
          <mat-label>{{'masterSetting.attributes.LabelName'| translate}}</mat-label>
          <input matInput formControlName="BA_CFF_LBL" autocomplete="off" />
        </mat-form-field>

        <div formArrayName="customformfield_values" class="col-lg-12">
          <div *ngIf="checkSelectedFieldType('Text Box')" [formGroupName]="0">
            <mat-form-field appearance="outline" color="primary">
              <mat-label>{{'masterSetting.attributes.DefaultValue'| translate}}</mat-label>
              <input type="text" matInput formControlName="CFF_VAL" (input)="customFieldSetDefaultValue('Text Box',0)" />
            </mat-form-field>
          </div>
          <div *ngIf="checkSelectedFieldType('Date Picker')" [formGroupName]="0">
            <mat-form-field appearance="outline">
              <mat-label>{{'masterSetting.attributes.ChooseDate'| translate}}</mat-label>
              <input matInput [matDatepicker]="datepicker" formControlName="CFF_VAL" (dateInput)="customFieldSetDefaultValue('Date Picker',0)" />
              <mat-datepicker-toggle matIconSuffix [for]="datepicker">
              </mat-datepicker-toggle>
              <mat-datepicker #datepicker></mat-datepicker>
              <button class="calendar-icon-btn"><img src="/assets/icons/calendar.svg" alt="img"/></button>
            </mat-form-field>
          </div>
          <div *ngIf="checkSelectedFieldType('Text Area')" [formGroupName]="0">
            <mat-form-field class="custom-textarea text-area-portion" appearance="outline">
              <mat-label>{{'masterSetting.attributes.DefaultValue'| translate}}</mat-label>
              <textarea matInput cdkTextareaAutosize #description cdkAutosizeMinRows="2" cdkAutosizeMaxRows="5"
                formControlName="CFF_VAL" (input)="customFieldSetDefaultValue('Text Area',0)"></textarea>

            </mat-form-field>
          </div>
          <div  *ngIf="disableOption('Dropdown') || disableOption('Dropdown Multiselect')">
            <div *ngFor="let field of customformfield_values?.controls;let i = index" [formGroupName]="i" class="grid-row-sub">
              <div class="field_part_check">
                <mat-form-field appearance="outline" color="primary">
                  <mat-label>{{'masterSetting.attributes.option'| translate}} {{ i + 1 }}</mat-label>
                  <input matInput formControlName="CFF_VAL" required="" />
                </mat-form-field>
                <ng-container *ngIf="customformfield_values?.controls?.length">
                  <mat-error *ngIf="submitted && customformfield_values.controls[i]?.get('CFF_VAL')?.hasError('required') && customformfield_values.controls[i]?.get('CFF_VAL')?.touched"   align="end" class="text-danger">
                    {{'loan.createLoan.thisFieldIsRequired'| translate}}
                  </mat-error>
                </ng-container>
              </div>
              <div class="grid-column-three check-area">
                <mat-checkbox [checked]="checkDefault(field)" (change)="updateCheckData(field, $event.checked, i)"
                  formControlName="isdefault">
                  <div style="font-size: 13px">
                    {{
                        checkSelectedFieldType("Dropdown Multiselect")
                          ? "Selected"
                          : "Make It Default"
                      }}
                  </div>
                </mat-checkbox>
              </div>
              <div  style="display: flex">
                <button (click)="addFieldValue()" *ngIf="i == customformfield_values.value.length - 1" mat-icon-button
                  matTooltip="Basic" class="sm-btn ml-auto">
                  <mat-icon>add_circle_outline</mat-icon>
                </button>
                <button  (click)="removeFieldValue(i)" *ngIf="i > -1 && i < customformfield_values.value.length - 1"
                  mat-icon-button matTooltip="Basic" class="sm-btn ml-auto">
                  <mat-icon>remove_circle_outline</mat-icon>
                </button>
              </div>
            </div>
          </div>
        </div>
        <div class="row_cls">

          <div class="check-list-opt">
            <div class="check-area" [ngStyle]="disableCheckbox() ? {'pointer-events':'none'}:{'pointer-events':'auto' }">
              <mat-checkbox [disabled]="disableCheckbox()"  class="example-margin" formControlName="isfilterable"></mat-checkbox>
              <span  [ngClass]="{'disable_txt':disableCheckbox()}">{{'masterSetting.attributes.FilterableField'| translate}}</span>
            </div> 
            <div class="check-area">
              <mat-checkbox  class="example-margin" formControlName="ismandatory"></mat-checkbox>
              <span>{{'masterSetting.attributes.MandatoryField'| translate}}</span>
            </div>    
          </div>
        </div>
        <mat-form-field appearance="outline" color="primary" class="common-gap">
          <mat-label>{{'customer.description'|translate}}</mat-label>
          <input matInput formControlName="BA_CFF_DE" />
        </mat-form-field>
      </form>
    </div>
    <div class="modal-footer">
      <div class="check-area">
        <mat-checkbox class="mr-auto" color="accent" [checked]="isAnother" (change)="isAnother = !isAnother">
        </mat-checkbox>
        <span>{{'common.createAnother' | translate}}</span>
      </div>
      <div class="modal_btn">
        <button class="cancel_btn" mat-dialog-close (click)="closeDialog()">{{'common.cancel' | translate}}</button>
        <button class="add_btn primary-btn" [disabled]="dataSubmitted"
          (click)="addCategoryFunction(isAnother)">{{ data?.selectedData ? ('common.save' | translate) : ('common.add' | translate) }}</button>
      </div>
    </div>
  </div>
</div>

---custom field dialog css----
::ng-deep {
    .ng-dropdown-panel .ng-dropdown-panel-items .ng-option .ng-tag-label {
        padding-right: 5px;
        font-size: 98%;
        font-weight: 400;
        color: #1AB1D2;
    }

    .disabled-checkbox {
        pointer-events: none;
        opacity: 0.5;

        .mat-mdc-form-field .mat-mdc-text-field-wrapper .mat-mdc-form-field-flex .mat-mdc-floating-label {
            pointer-events: none;
        }

        .mat-mdc-form-field .mat-mdc-text-field-wrapper .mat-mdc-form-field-flex .mat-mdc-floating-label.mdc-floating-label--float-above {
            pointer-events: none;
        }

        /* You can adjust the opacity value */
    }
}

@import '../../../../../../../host/src/assets/styles/variable.scss';

.add-color-modal {

    // width: 100%;
    // min-width: 600px;
    // overflow: hidden;
    // @media(max-width:991px){
    //     min-width: auto;
    //     max-width: 600px;
    // }
    .grid-row {
        display: flex;
        flex-wrap: wrap;
        margin: 0px -7.5px 0 -7.5px;

        .grid-column {
            flex-basis: 50%;
            max-width: 50%;
            padding: 0 7.5px;
            box-sizing: border-box;

            @media(max-width:767px) {
                flex-basis: 100%;
                max-width: 100%;
            }
        }
    }

    .close_btn {
        ::ng-deep {
            .mat-icon {
                color: rgba(0, 0, 0, 0.6);
            }
        }
    }

    .modal-body {
        height: 276px;
        overflow-y: auto;

        ::ng-deep {
            .ng-select {
                padding-bottom: 0;
            }

            .ng-value {
                border: none;
                border-radius: 0px;
                background-color: transparent !important;
            }

            .ng-input {
                input {
                    padding: 0 10px !important;
                    box-sizing: border-box !important;
                }
            }

            .ng-select .ng-select-container.ng-appearance-outline:after {
                border: none !important;

            }

            .ng-select.ng-select-single .ng-select-container.ng-appearance-outline .ng-clear-wrapper {
                bottom: 0px !important;
            }

            .ng-select.ng-select-single .ng-select-container.ng-appearance-outline .ng-arrow-wrapper {
                bottom: 3px !important;
            }
        }

        .row_cls {
            display: flex;
            flex-wrap: wrap;
            margin: 0 -5px;

            .col-lg-6 {
                flex-basis: 50%;
                max-width: 50%;
                padding: 0 5px;

                @media(max-width:991px) {
                    flex-basis: 100%;
                    max-width: 100%;
                }
            }
        }

    }

    .modal-footer {
        padding: 15px;
        border-top: 1px solid $border-color;
        display: flex;
        align-items: center;
        flex-wrap: wrap;
        justify-content: space-between;


    }

    .check-list-opt {
        display: flex;
        align-items: center;
        margin-bottom: 20px;

        >div {
            margin-right: 15px;
            margin-left: 8px;
        }
    }

    .check-area {
        display: inline-flex;
        align-items: center;
        justify-content: flex-start;

        span {
            color: $field-label;
            font-size: 14px;
            font-weight: 400;
            padding-left: 10px;
            line-height: 1.5;
        }

        ::ng-deep {
            .mdc-checkbox {
                padding: 0px;
            }

            .mdc-checkbox__background {
                top: 0;
                left: 0;
                border: 1px solid $checkbox !important;
                background-color: $white !important;
            }

            .mat-mdc-checkbox-checked {
                .mdc-checkbox__background {
                    border-color: $secondary-color !important;
                    background-color: $secondary-color !important;
                }
            }

            .mat-mdc-checkbox-touch-target {
                width: 35px;
                height: 35px;
            }
        }
    }



    .modal_btn {
        display: flex;
        align-items: center;



        button {
            margin-right: 10px;

        }

        .cancel_btn {
            border: 1px solid $secondary-color;
            box-shadow: none;

            &:hover {
                background-color: $secondary-color;
            }
        }

        .add_btn {
            width: auto;
            margin-top: 0;
            padding: 10px 18px;
            font-size: 12px;


        }
    }

    .text-area-portion {
        textarea {
            height: 36px !important;
            resize: none;
            min-height: auto !important;
        }

        ::ng-deep {
            .mat-mdc-text-field-wrapper {
                height: 56px !important;

                textarea {
                    height: 36px !important;
                    resize: none;
                }
            }

            .mat-mdc-form-field-infix {
                min-height: auto;
                height: 36px;
            }
        }
    }

    .grid-row-sub {
        display: flex;
        align-items: flex-start;
        justify-content: space-between;

    }

    .sm-btn {
        padding: 0;
        height: 27px;

        .mat-icon {
            color: $grey-color-seven;
        }
    }

    .disable_txt {
        opacity: 0.5;
    }

    .field_part_check {
        @media(max-width:575px) {
            width: 40%;
        }
    }
}
