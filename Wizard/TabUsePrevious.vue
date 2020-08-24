<template>
  <div style="padding: 140px">
    <div class="row">
      <button @click="getUsePrevious">
        Consultar
      </button>
    </div>

    <div class="row">
      <label>Fue evaluado en AT?</label><br>
      <input type="radio"
             name="radEvaluado"
             v-model="info.isEvaluatedFlag"
             :value="true">Si<br>
      <input type="radio"
             name="radEvaluado"
             v-model="info.isEvaluatedFlag"
             :value="false"> No<br>
      <br>
    </div>

    <div class="row">
      <label>Las necesidades evaluadas en AT:</label> <br>
      <input type="text"
             v-model="info.needsComments" >
    </div>


    <div class="row">
      <label>Indique otros servicios de AT que se le hayan provisto al estudiante especifique:</label> <br>
      <input type="text"
             v-model="info.othersServicesProvider" >
    </div>


    <div class="row">
      <label>El estudiante cuenta con equipo AT?</label>
      <input type="radio"
             name="radCuentaEquipoAT"
             v-model="info.studentWithEquipmentFlag"
             :value="true">Si<br>
      <input type="radio"
             name="radCuentaEquipoAT"
             v-model="info.studentWithEquipmentFlag"
             :value="false"> No<br>
    </div>

    <table class="table table-striped table-bordered">
      <thead>
        <tr>
          <th>Equipo de programa de AT</th>
          <th>Utilizacion / Satisfaccion</th>
          <th>Editar</th>
          <th>Eliminar</th>
        </tr>
      </thead>
      <tbody>
        <tr v-for="(item,index) in info.peiAtUsePreviousEquipments"
            :key="index">
          <th>{{ item.equipmentName }}</th>
          <th>
            <label>Utiliza el equipo regularmente?</label>
            <input type="radio"
                   v-model="item.frequencyEquipment"
                   :value="1">Si
            <input type="radio"
                   v-model="item.frequencyEquipment"
                   :value="0"> No<br>

            <label>Satisface adecuadamente la necesidad para su desempeño escolar?</label>
            <input type="radio"
                   v-model="item.performaceSchool"
                   :value="1">Si
            <input type="radio"
                   v-model="item.performaceSchool"
                   :value="0"> No
          </th>
          <!--<th>{{ item.equipmentName  }}</th>-->
          <!--<td>{{ item.reason }}</td>-->
          <td>
            <button @click="editEquipo(item)">
              Editar
            </button>
          </td>
          <td>
            <button @click="deleteEquipo(item.equipmentName)">
              Eliminar
            </button>
          </td>
        </tr>
      </tbody>
    </table>

    <br >



    <div class="row">
      <label>Equipo:</label> <br>
      <input type="text"
             v-model="equipo.equipmentName"
             id="txtEquipo" >
    </div>

    <div class="row">
      <label>Desde:</label> <br>
      <input type="text"
             v-model="equipo.fromDate"
             id="fechaInicio" >
    </div>

    <div class="row">
      <label>Utiliza el equipo regularmente?</label>
      <input type="radio"
             name="radNuevoEquipoAT"
             v-model="equipo.frequencyEquipment"
             :value="1">Si<br>
      <input type="radio"
             name="radNuevoEquipoAT"
             v-model="equipo.frequencyEquipment"
             :value="0"> No<br>
    </div>


    <div class="row">
      <label>Razon:</label> <br>
      <input type="text"
             v-model="equipo.reason"
             id="txtRazon" >
    </div>

    <div class="row">
      <label>Satisface adecuadamente la necesidad para su desempeño escolar?</label>
      <input type="radio"
             name="radNuevoSatisface"
             v-model="equipo.performaceSchool"
             :value="1">Si<br>
      <input type="radio"
             name="radNuevoSatisface"
             v-model="equipo.performaceSchool"
             :value="0"> No<br>
    </div>

    <div class="row">
      <label>Explique:</label> <br>
      <input type="text"
             v-model="equipo.explanation"
             id="txtExplique" >
    </div>

    <br >
    <div class="row">
      <button @click="addEquipo">
        Anadir
      </button>
      <button @click="postUsePrevius">
        Guardar
      </button>
    </div>
  </div>
</template>

<script>
    export default
        {
            data: () => ({
                mensaje: "como estas",
                info: {},
                equipo: {
                    "id": 0,
                    "peiAtUsePreviousId": 3,
                    "equipmentName": "",
                    "fromDate": "2018-05-17T00:00:00",
                    "frequencyEquipment": 1,
                    "reason": "",
                    "performaceSchool": 1,
                    "explanation": "",
                    "createdDate": "2019-08-09T10:02:09.533",
                    "createdById": "c1dba641-f31d-4878-82ca-be08ada0dcde"
                }
            }),
            methods: {
                getUsePrevious() {
                    this.$ApiGet(
                        "/api/pei/getpreviousequip?peiAtServiceId=6"
                    )
                        .then(res => {
                            this.info = res.data;

                        })
                        .catch(err => {
                            this.response = err;
                        });
                },
                postUsePrevius() {
                    this.$ApiPost(
                        "/api/pei/insertpreviousequip",
                        this.info
                    )
                    .catch(err => {
                        this.response = err;
                    });
                },
                addEquipo() {
                    let equipoReflejo = Object.assign({}, this.equipo);
                    this.info.peiAtUsePreviousEquipments.push(equipoReflejo);
                },
                deleteEquipo(itemRemove) {

                    let confirmacion = confirm("Esta seguro que desea eliminar el equipo/programa?");
                    if (confirmacion == true) {
                        let filteredItems = this.info.peiAtUsePreviousEquipments.filter(function (item) {
                            return item.equipmentName !== itemRemove
                        })
                        this.info.peiAtUsePreviousEquipments = filteredItems;
                    }


                },
                editEquipo(itemEdit) {
                    this.equipo = itemEdit;
                }
            }
        }

</script>