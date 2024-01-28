# starter-express-api

This is the simplest possible nodejs api using express that responds to any request with: 
```
const express = require('express');
const cors = require('cors');
const app = express();
const http = require('http');
const uuid = require('uuid');
const fs = require('fs')
const base64 = require('base64-js');
const path = require('path');

const { Expo } = require('expo-server-sdk');
const expo = new Expo();
const AWS = require('aws-sdk');
// ngrok tunnel --label edge=edghts_2aGIoeoQs4r33e6JsCsbGcZKeT5 http://localhost:80


const s3 = new AWS.S3({
  endpoint: 's3.wasabisys.com',
  accessKeyId: 'IFE2LYNKQ9R1RZ39UDYB',
  secretAccessKey: '6Nz7ypUtebn3yDYeZTLWenJ6AOdY7I3J9eHqKN5d',
  region:''
});
const dbNameUsersWasabi = 'hihellodbsusers';
const dbNameAdsWasabi = 'hihelloadsdbs';
const dbNameRoomWasabi = 'hihelloroomdbs';
const dbNameFileWasabi = 'hihellofiledbs';

require('dotenv').config();

// const port = process.env.PORT || 3100;
const port = 80;  

const detectCode = process.env.DETECT_LANGUAGE_CODE;
const clientKeyMDTP = process.env.CLIENT_KEY_MIDTRANS_PRODUCTION;
const serverKeyMDTP = process.env.SERVER_KEY_MIDTRANS_PRODUCTION;

// for midtrans
const midtransClient = require('midtrans-client');

// conection with front-end for useMethod / rout export
const UserRoutes = require('./routes/userRoute');

// detect language
const DetectLanguage = require('detectlanguage');
var detectlanguage = new DetectLanguage(detectCode);

app.use(cors());
app.use(express.json());
app.use(express.urlencoded({ extended: true }));
app.use(UserRoutes);

// midtren-client setup
const SnapApi = new midtransClient.Snap({
  isProduction: true,
  serverKey: serverKeyMDTP,
  clientKey: clientKeyMDTP
});

// Configuration of socket.io realTime
const server = http.createServer(app);
const { Server } = require('socket.io');

const io = new Server(server, {
  cors: {
    origin: 'http://localhost:3100',
    methods: ['GET', 'POST'],
  },
  maxHttpBufferSize: 10000000e8,
  pingTimeout: 60000

});

app.get('/check_lang_text/:text', async (req, res) => {
  // console.log(req.params.text)
  await detectlanguage.detect(req.params.text).then(result => {
    console.log(result)
    if (result.length == 0) {
      console.log('no lang');
      res.json('en');
    } else {
      let lang = result[0].language;
      res.json(lang);
    }
    // console.log(`Text: ${req.params.text}, Lang: ${lang}`);
  })
    ;
});
// for payment internet
// midtrans
// for notification transfer 
app.get('/notification_transfer/:orderId', async (req, res) => {
  // console.log(req.params.orderId);
  // get status of transaction that already recorded on midtrans (already `charge`-ed) 
  SnapApi.transaction.status(req.params.orderId)
    .then((response) => {
      res.json(response)
      console.log(response)
    })
    .catch(err => {
      console.log(`error to get information from id ${req.params.orderId}`)
    })
});
// cancel ads transaction
app.get('/cancel_ads_transaction/:oderId', async (req, res) => {
  await SnapApi.transaction.cancel(req.params.oderId)
    .then((response) => {
      // do something to `response` object
      // res.json(true);
      console.log(response);
    });
})

app.post('/payment-snap-credit', (req, res) => {
  // console.log(req.body);
  const randomUUID = uuid.v4();
  let parameter = {
    "transaction_details": {
      "order_id": randomUUID,
      "gross_amount": req.body.amount
    }, "credit_card": {
      "secure": true,
      "bank": "bni"
    }
  };
  SnapApi.createTransaction(parameter)
    .then((transaction) => {
      // transaction token
      let transactionToken = transaction.token;
      // transaction redirect url
      let transactionRedirectUrl = transaction.redirect_url;
      res.json({ orderId: randomUUID, token: transactionToken, url: transactionRedirectUrl })
    })
    .catch((e) => {
      console.log('Error occured:', e.message);
    });

})


const countriesDB = require('countries-db');

io.on('connection', (socket) => {
  console.log(`User connected ${socket.id}`);

  // -y
  socket.on('show_country_dbs', async (data) => {
    try {
      const allCountry = await countriesDB.getAllCountries();
      const allCountryArr = Object.keys(allCountry).map((key) => {
        // console.log(key);
        return allCountry[key]
      });
      socket.emit('recive_country_dbs', allCountryArr);
    } catch (error) {
      console.log(error)
    }
  });

  // -y
  socket.on('show_ads', async (data) => {
    // get data ads to show
    const getObjectData = async (key) => {
      const param = {
        Bucket: dbNameAdsWasabi,
        Key: key,
      };

      try {
        const data = await s3.getObject(param).promise();
        return JSON.parse(data.Body.toString());
      } catch (error) {
        console.error(`Error getting object '${key}':`, error);
        throw error;
      }
    };
    const getAllDataFromBucket = async () => {
      const param = {
        Bucket: dbNameAdsWasabi,
        MaxKeys: 30
      };
      try {
        const listObjectsResponse = await s3.listObjectsV2(param).promise();
        const objects = listObjectsResponse.Contents;
        const currentDate = new Date().getTime();
        const allData = await Promise.all(
          objects.map(async (object) => {
            const key = object.Key;
            const data = await getObjectData(key);
            return { key: key, data };
          })
        );
        const mapAllData = allData.map(dtbs => {
          const dataM = dtbs.data;
          return dataM
        });
        // get data user
        const params = {
          Bucket: dbNameUsersWasabi,
          Key: data.codeUserApp,
        };
        s3.getObject(params,async (err, dataR) => {
          if(err) {
            console.log(`Data doesn't exist !`, err)
          } else {
            const dataUser = JSON.parse(dataR.Body.toString());
            const mapAds = mapAllData.filter(dataAd => {    
              if(dataAd.times.timeEnds.timeMili == currentDate && dataAd.statePaymentText === 'settlement' || dataAd.times.timeEnds.timeMili == currentDate && dataAd.statePaymentText === 'success' || dataAd.statePaymentText === 'deny' || dataAd.statePaymentText === 'cancel' || dataAd.statePaymentText === 'expire') {
                      // delete file ad for image pick type
                if(dataAd.idFile !== '') {
                  const paramsFIle = {
                    Bucket: dbNameFileWasabi,
                    Key: dataAd.idFile,
                  };                                  
                  s3.deleteObject(paramsFIle, (err, dataM) => {
                    if (err) {
                      console.error('Error deleting object:', err);
                    } else {
                      console.log('File img Ad deleted successfully !');
                    }
                  });  
                }

                // delete file ad for video pick type
                if(dataAd.idFileV !== '' && dataAd.type === 'video') {
                  const paramsFIle = {
                    Bucket: dbNameFileWasabi,
                    Key: dataAd.idFileV,
                  };                                  
                  s3.deleteObject(paramsFIle, (err, dataM) => {
                    if (err) {
                      console.error('Error deleting object:', err);
                    } else {
                      console.log('File video Ad deleted successfully !');
                    }
                  });
                }
                      // delete ad
                const params = {
                  Bucket: dbNameAdsWasabi,
                  Key: dataAd.idAd,
                };                                     
                s3.deleteObject(params, (err, dataM) => {
                  if (err) {
                    console.error('Error deleting object:', err);
                  } else {
                    console.log('Ad deleted successfully !');
                  }
                });  
              } 
              return dataAd.times.timeEnds.timeMili != currentDate && dataAd.statePaymentText === 'settlement' || dataAd.times.timeEnds.timeMili != currentDate && dataAd.statePaymentText === 'success' || dataAd.statePaymentText !== 'deny' || dataAd.statePaymentText !== 'cancel' || dataAd.statePaymentText !== 'expire'
            });
            const filterBasedP1 = mapAds.filter(ad => {
              return ad.packagePic === 'p1' && ad.countries.country1 === dataUser.country
            }); 
            const filterBasedP2 = mapAds.filter(ad => {
              return ad.packagePic === 'p2' && ad.countries.country1 === dataUser.country || ad.countries.country2 === dataUser.country || ad.countries.country3 === dataUser.country || ad.countries.country4 === dataUser.country || ad.countries.country5 === dataUser.country
            }); 
            const filterBasedpAll = mapAds.filter(ad => {
              return ad.packagePic === 'all'
            });
            const combine1 = filterBasedP1.concat(filterBasedP2)
            const combineLast = filterBasedpAll.concat(combine1);
            // socket.emit('recive_advertisement_arr', {adsArr:combineLast});
            // console.log(combineLast)
            socket.emit('recive_advertisement_arr', {adsArr:[]});
          }
        });
        
      } catch (error) {
        console.log(error);
      }

    };
    getAllDataFromBucket();

  });

  // -y
  socket.on('get_data_advertisement_own', async (data) => {
    try {
      const getObjectData = async (key) => {
        const param = {
          Bucket: dbNameAdsWasabi,
          Key: key,
        };
        try {
          const data = await s3.getObject(param).promise();
          return JSON.parse(data.Body.toString());
        } catch (error) {
          console.error(`Error getting object '${key}':`, error);
          throw error;
        }
      };
      const getAllDataFromBucket = async () => {
        const params = {
          Bucket: dbNameAdsWasabi,
        };
        try {
          const listObjectsResponse = await s3.listObjectsV2(params).promise();
          const objects = listObjectsResponse.Contents;
          const allData = await Promise.all(
            objects.map(async (object) => {
              const key = object.Key;
              const data = await getObjectData(key);
              return { key: key, data };
            })
          );
          const mapAllData = allData.map(dtbs => {
            const dataM = dtbs.data
            return dataM
          });
          const filterAdsDbs = mapAllData.filter(ads => {
            return ads.codeUser === data.codeUser
          });
          socket.emit('recive_data_advertisement_own', { adsCollection: filterAdsDbs});
        } catch (error) {
          console.error('Error listing objects:', error);
        }
      };
      getAllDataFromBucket();
    } catch (error) {
      console.log(error)
    }
  });

  // log in user -y
  socket.on('check_acount_log_in', async (data) => {
    try {
      const getObjectData = async (key) => {
        const params = {
          Bucket: dbNameUsersWasabi,
          Key: key,
        };
        try {
          const data = await s3.getObject(params).promise();
          return JSON.parse(data.Body.toString());
        } catch (error) {
          console.error(`Error getting object '${key}':`, error);
          throw error;
        }
      };
      const getAllDataFromBucket = async () => {
        const params = {
          Bucket: dbNameUsersWasabi,
        };
        try {
          const listObjectsResponse = await s3.listObjectsV2(params).promise();
          const objects = listObjectsResponse.Contents;
          const allData = await Promise.all(
            objects.map(async (object) => {
              const key = object.Key;
              const data = await getObjectData(key);
              return { key: key, data };
            })
          );
          // if (allData.length == 0) {
          //   console.log('database is empty, no data yet !');
          // } else {
            // console.log(`amount of data :${allData.length}`);
            const mapAllData = allData.map(dtbs => {
              const dataM = dtbs.data
              return dataM
            })
            const findData = mapAllData.find(dataU => {
              return dataU.codeShare === data.codeShare
            });
            if (findData != undefined) {
              // const dataUser = findData.data;
              socket.emit('recive_check_acount_log_in', { dataUser: findData})
            } else {
              socket.emit('recive_check_acount_log_in', { dataUser: null })
            }
          // }
          //   console.log('All data as JSON:',allData)
        } catch (error) {
          console.error('Error listing objects:', error);
        }
      };

      getAllDataFromBucket();
    } catch (error) {
      console.log(error)
    }

  });

  // join userApp_data -y
  socket.on('userJoinApp_data', async (data) => {
    try {
      const randomUUID = uuid.v4()
      // get all data to check if user is already exist
      const getObjectData = async (key) => {
        const params = {
          Bucket: dbNameUsersWasabi,
          Key: key,
        };

        try {
          const data = await s3.getObject(params).promise();
          return JSON.parse(data.Body.toString());
        } catch (error) {
          console.error(`Error getting object '${key}':`, error);
          throw error;
        }
      };
      const getAllDataFromBucket = async () => {
        const params = {
          Bucket: dbNameUsersWasabi,
        };
        try {
          const listObjectsResponse = await s3.listObjectsV2(params).promise();
          const objects = listObjectsResponse.Contents;
          const allData = await Promise.all(
            objects.map(async (object) => {
              const key = object.Key;
              const data = await getObjectData(key);
              return { key: key, data };
            })
          );

          const mapAllData = allData.map(data => {
            const dataR = data.data;
            return dataR
          })
          const findData = mapAllData.find(dataU => {
            return dataU.codeShare === data.codeShare
          });
          if (findData == undefined) {
            const randomUUIDFIle = uuid.v4();
            // add new user
            const dataForm = data.dataForm
            dataForm.code = `${dataForm.codeShare}-${randomUUID}`;
            dataForm.codeShare = dataForm.codeShare;
            // add file , get file url and add user
            let keyFile =  `img-prof-${randomUUIDFIle}`;
            const params = {
              Bucket: dbNameFileWasabi,
              Key: keyFile,
              Body: Buffer.from(data.image, 'base64'),
              ACL: 'public-read', // Opsional, untuk memberikan akses publik
            };
            s3.upload(params, (err, data) => {
              if (err) {
                console.error('Error uploading file:', err);
              } else {
                console.log('File uploaded successfully:', data.Location);
                const expirationTime = 31536000;
                const signedUrl = s3.getSignedUrl('getObject', {
                  Bucket: dbNameFileWasabi,
                  Key: keyFile,
                  Expires: expirationTime,
                });
                  dataForm.imgProfileUser = signedUrl;
                  dataForm.idImgProf = keyFile;
                  // add user
                  let keyDataUser = `${dataForm.codeShare}-${randomUUID}`;
                  const params = {
                    Bucket: dbNameUsersWasabi,
                    Key: keyDataUser,
                    Body: JSON.stringify(dataForm), // Mengonversi objek JSON menjadi string JSON
                    ContentType: 'application/json', // Tentukan jenis konten sebagai JSON
                  };
                  s3.putObject(params, (err, dataK) => {
                    if (err) {
                      console.error('Error adding JSON object:', err);
                    } else {
                      console.log('added user successfully:', dataK);
                      socket.emit('recive_err_register', { state: true, msgSuccess: `Regstration is success 🥳 !`, codeUser: dataForm.code })
                    }
                  });
              }
            });

          } else {
            const dataUser = findData
            socket.emit('recive_err_register', { state: false, msgErr: `Filed regidteration`, codeUser: dataUser.code, msgSuggestion: 'Change with another code or another combination from code country and phone number( you can enter any number ).' })
          }
        } catch (error) {
          console.error('Error listing objects:', error);
        }
      };
      getAllDataFromBucket();
    } catch (error) {
      console.log(error)
    }
  });

  //getFullDataUser -y
  socket.on('getFull_dataUser', async code => {
    // console.log(code);
    try {
      if (code) {

        // get data use stream
        const params = {
          Bucket: dbNameUsersWasabi,
          Key: code
        };
        function getObjectStream(params) {
          return s3.getObject(params).createReadStream();
        }
        const chunks = [];
        getObjectStream(params)
          .on('data', function (chunk) {
            // Proses setiap bagian data yang diambil
            chunks.push(chunk);
          })
          .on('end', function () {
            // Lakukan sesuatu dengan data yang telah digabungkan
            const combinedDataC = combineChunks(chunks);
            const fileData = JSON.parse(combinedDataC.toString());
            console.log(fileData)
            socket.emit('recive_fullDataUser', fileData)
          })
          .on('error', function (err) {
            // Tangani kesalahan jika ada
            console.error('Error:', err);
          });
        // Implementasikan fungsi untuk menggabungkan bagian-bagian data yang telah dipecah
        function combineChunks(chunks) {
          // Gabungkan bagian-bagian data menjadi satu
          return Buffer.concat(chunks);
        }
      }
      // console.log(code);
    } catch (error) {
      console.log(error)
    }
  });

  // getFullDataUser other -y
  socket.on('getFull_dataUser_other', async code => {
    try {
      // console.log(code);
      if (code) {
        const params = {
          Bucket: dbNameUsersWasabi,
          Key: code,
        };
        function getObjectStream(params) {
          return s3.getObject(params).createReadStream();
        }
        const chunks = [];
        getObjectStream(params)
          .on('data', function (chunk) {
            // Proses setiap bagian data yang diambil
            chunks.push(chunk);
          })
          .on('end', function () {
            // Lakukan sesuatu dengan data yang telah digabungkan
            const combinedDataC = combineChunks(chunks);
            const fileData = JSON.parse(combinedDataC.toString());
            socket.emit('recive_fullDataUser_other', fileData);
          })
          .on('error', function (err) {
            // Tangani kesalahan jika ada
            console.error('Error:', err);
          });
        // Implementasikan fungsi untuk menggabungkan bagian-bagian data yang telah dipecah
        function combineChunks(chunks) {
          // Gabungkan bagian-bagian data menjadi satu
          return Buffer.concat(chunks);
        }
      }
    } catch (error) {
      console.log(error)
    }

  });

  // addContact -y
  socket.on('addContact_data', async (data) => {
    try {
      const getObjectData = async (key) => {
        const params = {
          Bucket: dbNameUsersWasabi,
          Key: key,
        };
        try {
          const data = await s3.getObject(params).promise();
          return JSON.parse(data.Body.toString());
        } catch (error) {
          console.error(`Error getting object '${key}':`, error);
          throw error;
        }
      };
      const getAllDataFromBucket = async () => {
        const params = {
          Bucket: dbNameUsersWasabi,
        };
        try {
          const listObjectsResponse = await s3.listObjectsV2(params).promise();
          const objects = listObjectsResponse.Contents;
          const allData = await Promise.all(
            objects.map(async (object) => {
              const key = object.Key;
              const data = await getObjectData(key);
              return { key: key, data };
            })
          );
          const mapAllData = allData.map(dtbs => {
            const dataR = dtbs.data;
            return dataR
          });

          //   get data user
          // console.log(allData)
          const dataUser = mapAllData.find(dataDbs => {
            return dataDbs.code === data.codeUser
          });
          const userContactColl = dataUser.contactCollections;
          //   get data contact
          const dataContact = mapAllData.find(dataDbs => {
            return dataDbs.codeShare === data.inputCode
          });
          // adding data function
          const addingData = async () => {
            const contactCollContact = dataContact.contactCollections;
            // ceck contack is already exist in contact collection
            const ceckContact = contactCollContact.find(dataCntk => {
              return dataCntk.roomPath === `${dataUser.code}-${dataContact.code}`
            });
            userContactColl.push({
              roomPath: `${dataUser.code}-${dataContact.code}`,
              code: dataContact.code,
              codeShare: data.inputCode,
              name: data.callName,
              imgProfCntk: dataContact.imgProfileUser,
              idImgProfile: dataContact.idImgProf,
              mimeImgProf: dataContact.mimeImgProf,
              msgStates: 0,
              tokenDevice: dataContact.tokenDevice,
              gender:dataContact.genderUser
            });
            contactCollContact.push({
              roomPath: `${dataUser.code}-${dataContact.code}`,
              code: data.codeUser,
              codeShare: dataUser.codeShare,
              name: '',
              imgProfCntk:dataUser.imgProfileUser,
              idImgProfile: dataUser.idImgProf,
              mimeImgProf: dataUser.mimeImgProf,
              msgStates: 0,
              tokenDevice: dataUser.tokenDevice,
              gender:dataUser.genderUser
            });
            // update for user
            const params = {
              Bucket: dbNameUsersWasabi,
              Key: dataUser.code,
            };
            s3.getObject(params, (err, dataR) => {
              if (err) {
                console.error('Error getting object:', err);
              } else {
                const existingData = JSON.parse(dataR.Body.toString());
                existingData.contactCollections = userContactColl
                const updatedParams = {
                  Bucket: dbNameUsersWasabi,
                  Key: data.codeUser,
                  Body: JSON.stringify(existingData),
                  ContentType: 'application/json',
                };
                s3.putObject(updatedParams, (err, dataK) => {
                  if (err) {
                    console.error('Error updating object:', err);
                  } else {
                    console.log('successfuly updated data user (add contact):', dataK);
                  }
                });
              }
            });
            if (ceckContact == undefined) {
              console.log('data belum ada');
              // for contact
              const params = {
                Bucket: dbNameUsersWasabi,
                Key: dataContact.code,
              };
              s3.getObject(params, (err, dataK) => {
                if (err) {
                  console.error('Error getting object:', err);
                } else {
                  const existingData = JSON.parse(dataK.Body.toString());
                  existingData.contactCollections = contactCollContact;
                  const updatedParams = {
                    Bucket: dbNameUsersWasabi,
                    Key: dataContact.code,
                    Body: JSON.stringify(existingData),
                    ContentType: 'application/json',
                  };
                  s3.putObject(updatedParams, (err, dataK) => {
                    if (err) {
                      console.error('Error updating object:', err);
                    } else {
                      console.log('successfuly updated data contact(addcontact):', dataK);
                    }
                  });


                }
              });
            }
            //         // update for user contact 
            socket.broadcast.emit('update_newConttactColl', { contactColl: contactCollContact, codeContact: dataContact.code });
          }
          // ceck status contact
          if (dataContact != undefined) {
            const findSameContact = userContactColl.find(dataCntk => {
              return dataCntk.code === dataContact.code;
            });
            // console.log(dataContact.code)
            if (dataContact != null && dataContact.code !== data.codeUser && findSameContact == undefined) {
              socket.emit('send_state_add_contact', { state: true, msgSuccess: `Success add contact with name : ${data.callName} with code : ${data.inputCode}👌` });
              addingData()

            }
            if (findSameContact != undefined) {
              socket.emit('send_state_add_contact', {
                state: false,
                msgErr: `Contact with code: ${data.inputCode} is already exist 😅 ! `,
                msgSuggestion: `Enter with another code 😊 !`
              });
            }
            if (dataContact.code === data.codeUser) {
              socket.emit('send_state_add_contact', {
                state: false,
                msgErr: `Can't fill with own code 😓 !`,
                msgSuggestion: `Enter with another code 😊 !`
              });
            }
          } else {
            // console.log(`user with code ... doesn't exist`)
            socket.emit('send_state_add_contact', {
              state: false,
              msgErr: `User with code: ${data.inputCode} doesn't exist 😓`,
              msgSuggestion: `Enter with the correct code 😁 !`
            });
          }
        } catch (error) {
          console.error('Error listing objects:', error);
        }
      };
      getAllDataFromBucket();

    } catch (error) {
      console.log(error)
    }
  });

  // announcment alert -y
  socket.on('announcement_alert', (data) => {
    try {
      socket.emit('recive_alert', data);
    } catch (error) {
      console.log(error)
    }
  })

  // get full database users app -y
  socket.on('get_fullDataBase_usersApp', async (data) => {
    try {
      const getObjectData = async (key) => {
        const params = {
          Bucket: dbNameUsersWasabi,
          Key: key,
        };
        try {
          const data = await s3.getObject(params).promise();
          return JSON.parse(data.Body.toString());
        } catch (error) {
          console.error(`Error getting object '${key}':`, error);
          throw error;
        }
      };
      const getAllDataFromBucket = async () => {
        const params = {
          Bucket: dbNameUsersWasabi,
        };
        try {
          const listObjectsResponse = await s3.listObjectsV2(params).promise();
          const objects = listObjectsResponse.Contents;
          const allData = await Promise.all(
            objects.map(async (object) => {
              const key = object.Key;
              const data = await getObjectData(key);
              return { key: key, data };
            })
          );
          const mapAllData = allData.map(dtbs => {
            const dataM = dtbs.data
            return dataM
          });
          socket.emit('recive_fullDataBase_usersApp', mapAllData);

        } catch (error) {
          console.error('Error listing objects:', error);
        }
      };

      getAllDataFromBucket();

    } catch (error) {
      console.log(error)
    }
  });

  socket.on('sendData_to_page', (data) => {
    // console.log(data)
    try {
      socket.emit('reciveData_to_page', data);
    } catch (error) {
      console.log(error)
    }
  });
  // show users per country -y
  socket.on('show_country_users', async (data) => {
    try {
      const getObjectData = async (key) => {
        const params = {
          Bucket: dbNameUsersWasabi,
          Key: key,
        };
        try {
          const data = await s3.getObject(params).promise();
          return JSON.parse(data.Body.toString());
        } catch (error) {
          console.error(`Error getting object '${key}':`, error);
          throw error;
        }
      };
      const getAllDataFromBucket = async () => {
        const params = {
          Bucket: dbNameUsersWasabi,
        };
        try {
          const listObjectsResponse = await s3.listObjectsV2(params).promise();
          const objects = listObjectsResponse.Contents;
          const allData = await Promise.all(
            objects.map(async (object) => {
              const key = object.Key;
              const data = await getObjectData(key);
              return { key: key, data };
            })
          );
          const mapAllData = allData.map(dtbs => {
            const dataM = dtbs.data
            return dataM
          });
          const getNameCountry = data.map(dataCountry => {
            const getUsersCounrty = mapAllData.filter(userData => {
              return userData.country === dataCountry.name
            });
            return dataCountry.name
          });
          // const getCountry = 
          const getNew = getNameCountry.map(country => {
            const getUsersCounrty = mapAllData.filter(userData => {
              return userData.country === country;
            });
            const objDataStatisticCountry = {
              country: country,
              users: getUsersCounrty.length
            }
            return objDataStatisticCountry
          })
          socket.emit('recive_show_country_users', getNew)

        } catch (error) {
          console.error('Error listing objects:', error);
        }
      };

      getAllDataFromBucket();

    } catch (error) {
      console.log(error)
    }
  });
  // send msg -y
  socket.on('send_msg', async (data) => {
    try {
      // console.log(data)
      const params = {
        Bucket: dbNameRoomWasabi,
        Key: data.roomPath,
      };
      s3.getObject(params, (err, dataR) => {
        if (err) {
          console.log(`Data doesn't exist !`, err)
        } else {
          const dataRoom = JSON.parse(dataR.Body.toString());
          //   get data user
          const params = {
            Bucket: dbNameUsersWasabi,
            Key: data.codeUser,
          };
          s3.getObject(params, async (err, dataK) => {
            if (err) {
              console.log(`Data doesn't exist !`, err)
            } else {
              const dataUser = JSON.parse(dataK.Body.toString());
              // add msg
              if (data.dataMsg.type === 'text') {
                if (data.dataMsg.message !== "") {
                  await detectlanguage.detect(data.dataMsg.message).then(result => {
                    if (result.length != 0) {

                      let lang = result[0].language;
                      data.dataMsg.langMsg = lang
                    } else {
                      data.dataMsg.langMsg = "en";
                    }
                  });
                } else {
                  data.dataMsg.langMsg = "";
                }
                dataRoom.message.push(data.dataMsg);
                // update room
                const updatedParams = {
                  Bucket: dbNameRoomWasabi,
                  Key: data.roomPath,
                  Body: JSON.stringify(dataRoom),
                  ContentType: 'application/json',
                };

                s3.putObject(updatedParams, (err, dataX) => {
                  if (err) {
                    console.error('Error updating object:', err);
                  } else {
                    console.log('successfuly updated data room:', dataX);
                  }
                });

              } else {
                // console.log(data)
                const randomUUID = uuid.v4();
                let keyFile = `msg-file-${randomUUID}`;
                const params = {
                  Bucket: dbNameFileWasabi,
                  Key: keyFile,
                  Body: Buffer.from(data.base64data, 'base64'),
                  ACL: 'public-read', // Opsional, untuk memberikan akses publik
                };
                s3.upload(params, (err, dataR) => {
                  if (err) {
                    console.error('Error uploading file:', err);
                  } else {
                    console.log('File uploaded successfully:', dataR.Location);
                    const expirationTime = 1814400;
                    const signedUrl = s3.getSignedUrl('getObject', {
                      Bucket: dbNameFileWasabi,
                      Key: keyFile,
                      Expires: expirationTime,
                    });
                    // console.log(signedUrl);
                    if (data.dataMsg.type === 'image') {
                      data.dataMsg.messageImage = signedUrl
                    } else if (data.dataMsg.type === 'video') {
                      data.dataMsg.messageVideo = signedUrl
                    } else if (data.dataMsg.type === 'file') {
                      data.dataMsg.messageFile = signedUrl
                    } else {
                      data.dataMsg.messageAudio = signedUrl
                    }
                    data.dataMsg.idFile = keyFile;
                    dataRoom.message.push(data.dataMsg);
                    // update room
                    const updatedParams = {
                      Bucket: dbNameRoomWasabi,
                      Key: data.roomPath,
                      Body: JSON.stringify(dataRoom),
                      ContentType: 'application/json',
                    };
                    s3.putObject(updatedParams, (err, dataB) => {
                      if (err) {
                        console.error('Error updating object:', err);
                        if(data.dataMsg.type === 'audio') {
                          socket.emit('status_send_msg', {status:false})
                        }
                      } else {
                        console.log('successfuly updated data room !', dataRoom);
                        if(data.dataMsg.type === 'audio') {
                          socket.emit('status_send_msg', {status:true})
                        }
                      }
                    });
                    
                  }
                });
              }
              // do something to contact or group
              if (data.type === 'contact') {
                const msgColl = dataRoom.message;
                const contactColl = dataUser.contactCollections
                const getContact = contactColl.find(contact => {
                  return contact.roomPath === data.roomPath
                });
                // get data contact
                const params = {
                  Bucket: dbNameUsersWasabi,
                  Key: getContact.code,
                };
                s3.getObject(params, (err, dataK) => {
                  if (err) {
                    console.log(`Data doesn't exist !`)
                  } else {
                    const dataContact = JSON.parse(dataK.Body.toString());
                    const contactCollCntc = dataContact.contactCollections;
                    const msgStates = dataContact.msgStatuses;
                    const filterMsgOtherForOther = msgColl.filter(msg => {
                      return msg.code !== dataContact.code && msg.state == false
                    });
                    const mapCnct = contactCollCntc.map(contact => {
                      if (contact.roomPath === data.roomPath) {
                        contact.msgStates = filterMsgOtherForOther.length
                      }
                      return contact
                    });
                    // update data contact
                    dataContact.contactCollections = mapCnct;
                    dataContact.msgStates = msgStates + filterMsgOtherForOther
                    const updatedParams = {
                      Bucket: dbNameUsersWasabi,
                      Key: dataContact.code,
                      Body: JSON.stringify(dataContact),
                      ContentType: 'application/json',
                    };
                    s3.putObject(updatedParams, (err, dataZ) => {
                      if (err) {
                        console.error('Error updating object:', err);
                      } else {
                        console.log('successfuly updated data contact !');
                        socket.emit('notification_alrt', { sender: getContact.callName !== '' ? getContact.callName : getContact.codeShare, type: data.dataMsg.type, msg: data.dataMsg.type === 'text' ? data.dataMsg.message : '', codeUser: getContact.code })
                      }
                    });
                  }
                });
              } else {
                const msgColl = dataRoom.message;
                const groupMember = dataRoom.groupMember;
                groupMember.forEach(async member => {
                  // get data user each member of group
                  const params = {
                    Bucket: dbNameUsersWasabi,
                    Key: member.codeR,
                  };
                  s3.getObject(params, (err, dataZ) => {
                    if (err) {
                      console.log(`Data doesn't exist !`)
                    } else {
                      const memberData = JSON.parse(dataZ.Body.toString());
                      const groupColl = memberData.groupCollections;
                      const msgStates = memberData.msgStatuses
                      const filterMsg = msgColl.filter(msg => {
                        return msg.code !== memberData.code && msg.state == false
                      });
                      const mapingGroup = groupColl.map(group => {
                        if (group.room === memberData.roomPath) {
                          group.msgStatuses = group.msgStatuses + 1
                        }
                        return group
                      });
                      memberData.groupCollections = mapingGroup;
                      memberData.msgStatuses = msgStates + filterMsg.length
                      // update data user each member
                      const updatedParams = {
                        Bucket: dbNameUsersWasabi,
                        Key: member.codeR,
                        Body: JSON.stringify(memberData),
                        ContentType: 'application/json',
                      };
                      s3.putObject(updatedParams, (err, dataS) => {
                        if (err) {
                          console.error('Error updating object:', err);
                        } else {
                          socket.to(data.roomPath).emit("recive_message", { msg: data.dataMsg, typePage: 'group' });
                          socket.emit('send_msg_self', { msgColl: dataRoom.message, msg: data.dataMsg, typePage: 'group' });
                        }
                      });
                    }

                  });

                });
              }
            }
          });
        }
      });
    } catch (error) {
      console.log(error)
    }
  });
  // -y
  socket.on('send_self', async (data) => {
    // console.log(data);
    try {
      socket.emit('recive_send_self', data.msg);
    } catch (error) {
      console.log(error)
    }
  });
  //-y 
  socket.on('send_msg_other', (data) => {
    try {
      // console.log(data.msg, data.room);
      socket.to(data.room).emit('recive_send_other', data.msg);
    } catch (error) {
      console.log(error)
    }
  });

  // for test value 
  socket.on('res', async data => {
    try {
      console.log(data);
    } catch (error) {
      console.log(error)
    }
  });

  // delete msg -y
  socket.on('delete_msg', async (data) => {
    try {
      const params = {
        Bucket: dbNameRoomWasabi,
        Key: data.roomPath,
      };
      s3.getObject(params, (err, dataR) => {
        if (err) {
          console.log(`Data doesn't exist !`, err)
        } else {
          const dataRoom = JSON.parse(dataR.Body.toString());
          const messageColl = dataRoom.message;
          const filterMessage = messageColl.filter(msg => {
            return msg.timeSandComplete !== data.msgData.timeSandComplete
          });
          //   updating 
          if (dataRoom.type === 'contact') {
            const params = {
              Bucket: dbNameUsersWasabi,
              Key: data.codeUserApp,
            };
            s3.getObject(params, (err, dataX) => {
              if (err) {
                console.log(`Data doesn't exist !`, err)
              } else {
                const dataUser = JSON.parse(dataX.Body.toString());
                const contactColl = dataUser.contactCollections
                const findContact = contactColl.find(cntc => {
                  return cntc.roomPath === data.roomPath
                });
                //  get data user for contact
                const params = {
                  Bucket: dbNameUsersWasabi,
                  Key: findContact.code,
                };
                s3.getObject(params, (err, dataB) => {
                  if (err) {
                    console.log(`Data doesn't exist !`, err)
                  } else {
                    const dataContact = JSON.parse(dataB.Body.toString());
                    const getCntcColl = dataContact.contactCollections;
                    const mapingCntcColl = getCntcColl.map(cntc => {
                      if (cntc.roomPath === data.roomPath) {
                        cntc.msgStates = cntc.msgStates - 1
                      }
                      return cntc
                    });
                    //   update data user for contact
                    dataContact.contactCollections = mapingCntcColl;
                    dataContact.msgStatuses = dataContact.msgStatuses - 1
                    const updatedParams = {
                      Bucket: dbNameUsersWasabi,
                      Key: findContact.code,
                      Body: JSON.stringify(dataContact),
                      ContentType: 'application/json',
                    };
                    s3.putObject(updatedParams, (err, dataZ) => {
                      if (err) {
                        console.error('Error updating object:', err);
                      } else {
                        console.log('successfuly updated data !',);
                      }
                    });

                  }
                });
              }
            });
          } else {
            const groupMember = dataRoom.groupMember;
            groupMember.map(async member => {
              const params1 = {
                Bucket: dbNameUsersWasabi,
                Key: member.codeR
              };
              s3.getObject(params1, (err, data) => {
                if (err) {
                  console.error('Error getting object:', err);
                } else {
                  const dataMember = JSON.parse(data.Body.toString());
                  const msgStatuses = dataMember.msgStatuses
                  const groupColl = dataMember.groupCollections;
                  if (dataMember.code !== data.codeUserApp) {
                    const mapingGroup = groupColl.map(group => {
                      if (group.room === data.roomPath) {
                        group.msgStatuses = group.msgStatuses - 1
                      }
                      return group
                    });
                    // update data group
                    dataMember.groupCollections = mapingGroup;
                    dataMember.msgStatuses = msgStatuses - 1
                    const updatedParams = {
                      Bucket: dbNameUsersWasabi,
                      Key: dataMember.code,
                      Body: JSON.stringify(dataMember),
                      ContentType: 'application/json',
                    };
                    s3.putObject(updatedParams, (err, dataM) => {
                      if (err) {
                        console.error('Error updating object:', err);
                      } else {
                        console.log('successfuly updated data !');
                      }
                    });
                  }
                }
                });
              // if (dataUser.code !== data.codeUserApp) {
              //   const mapingGroup = groupColl.map(group => {
              //     if (group.room === data.roomPath) {
              //       group.msgStatuses = group.msgStatuses - 1
              //     }
              //     return group
              //   });
              //   // update data group
              //   dataUser.groupCollections = mapingGroup;
              //   dataUser.msgStatuses = dataUser.msgStatuses - 1
              //   const updatedParams = {
              //     Bucket: dbNameUsersWasabi,
              //     Key: dataUser.code,
              //     Body: JSON.stringify(dataUser),
              //     ContentType: 'application/json',
              //   };

              //   s3.putObject(updatedParams, (err, dataM) => {
              //     if (err) {
              //       console.error('Error updating object:', err);
              //     } else {
              //       console.log('successfuly updated data:', dataM);
              //     }
              //   });
              // }
            });
          };
          //   update data room
          dataRoom.message = filterMessage;
          const updatedParams = {
            Bucket: dbNameRoomWasabi,
            Key: data.roomPath,
            Body: JSON.stringify(dataRoom),
            ContentType: 'application/json',
          };
          s3.putObject(updatedParams, (err, dataN) => {
            if (err) {
              console.error('Error updating object:', err);
            } else {
              console.log('successfuly updated data room !');
              socket.to(data.roomPath).emit('response_delete_msg', filterMessage);
              socket.emit('response_self', filterMessage);
            }
          });

        }
      });


    } catch (error) {
      console.log(error)
    }
  });

  socket.on('save_edit_profile', async (data) => {
    try {

      const params = {
        Bucket: dbNameUsersWasabi,
        Key: data.codeUser,
      };

      s3.getObject(params, (err, dataR) => {
        if (err) {
          console.error('Error getting object:', err);
        } else {
          const existingData = JSON.parse(dataR.Body.toString());
          // Misalnya, kita akan menambahkan properti baru atau memperbarui yang sudah ada
          existingData.name = data.dataChange.name;
          existingData.email = data.dataChange.email;
          existingData.genderUser = data.dataChange.genderUser;
          existingData.birthYear = data.dataChange.birthYear;
          existingData.country = data.dataChange.country;
          existingData.countryUser = data.dataChange.countryUser;
          existingData.YtbUrl = data.dataChange.YtbUrl;
          existingData.IgUrl = data.dataChange.IgUrl;
          existingData.TweetUrl = data.dataChange.TweetUrl;
          existingData.WebUrl = data.dataChange.WebUrl;
          const updatedParams = {
            Bucket: dbNameUsersWasabi,
            Key: data.codeUser,
            Body: JSON.stringify(existingData),
            ContentType: 'application/json',
          };

          s3.putObject(updatedParams, (err, dataX) => {
            if (err) {
              console.error('Error updating object:', err);
            } else {
              console.log('successfuly updated data user:', dataX);
            }
          });

        }
      });

      // change for contact user

      socket.emit('get_response_update');
      // console.log(updateUserDbs);
    } catch (error) {
      console.log(error)
    }
  });

  // -y
  socket.on('change_img_profile', async (data) => {
    try {
      const params = {
        Bucket: dbNameUsersWasabi,
        Key: data.codeUser,
      };
      s3.getObject(params, async (err, dataR) => {
        if (err) {
          console.error('Error getting object:', err);
        } else {
          const randomUUID = uuid.v4();
          const existingData = JSON.parse(dataR.Body.toString());
          const contactColl = existingData.contactCollections
          let keyFile = `img-prof-${randomUUID}.${data.fileExt}`;
          const params = {
            Bucket: dbNameFileWasabi,
            Key: keyFile,
            Body: Buffer.from(data.base64data, 'base64'),
            ACL: 'public-read', // Opsional, untuk memberikan akses publik
          };
          s3.upload(params, (err, dataK) => {
            if (err) {
              console.error('Error uploading file:', err);
            } else {
              console.log('img profile uploaded successfully:', dataK.Location);
              const expirationTime = 31536000;
              const signedUrl = s3.getSignedUrl('getObject', {
                Bucket: dbNameFileWasabi,
                Key: keyFile,
                Expires: expirationTime,
              });
              // update user data
              existingData.imgProfileUser = signedUrl;
              existingData.idImgProf = keyFile;
              const updatedParams = {
                Bucket: dbNameUsersWasabi,
                Key: data.codeUser,
                Body: JSON.stringify(existingData),
                ContentType: 'application/json',
              };
              s3.putObject(updatedParams, (err, dataK) => {
                if (err) {
                  console.error('Error updating object:', err);
                } else {
                  console.log('successfuly updated img profile:', dataK);
                }
              });
              // update each contact
              contactColl.forEach(contact => {
                const getObjectP = {
                  Bucket: dbNameUsersWasabi,
                  Key: contact.code,
                };
                s3.getObject(getObjectP, (err, dataK) => {
                  if (err) {
                    console.log('error')
                  } else {
                    const dataContact = JSON.parse(dataK.Body.toString());
                    const contactColl = dataContact.contactCollections;
                    const dltColl = dataContact.deleteContactCollections;
                    contactColl.map(cntc => {
                      if (cntc.code === data.codeUser) {
                        cntc.idImgProfile = keyFile;
                        cntc.imgProfCntk = signedUrl
                      }
                      return cntc
                    });
                    dltColl.map(cntc => {
                      if (cntc.codeR === data.codeUser) {
                        cntc.idImgProfile = keyFile;
                        cntc.imgProfCntk = signedUrl
                      }
                      return cntc
                    });
                    dataContact.contactCollections = contactColl;
                    dataContact.deleteContactCollections = dltColl;
                    // update data contact
                    const updatedParams = {
                      Bucket: dbNameUsersWasabi,
                      Key: dataContact.code,
                      Body: JSON.stringify(dataContact),
                      ContentType: 'application/json',
                    };
                    s3.putObject(updatedParams, (err, dataK) => {
                      if (err) {
                        console.error('Error updating object:', err);
                      } else {
                        console.log('successfuly updated img profile coll contact:', dataK);
                      }
                    });
                  }
                });
              });
                }
              });
        }
      });
      // console.log(updateForMe);    
    } catch (error) {
      console.log(error)
    }
  });

  // edit contact -y
  socket.on('save_edit_contact', async (data) => {
    try {
      const params = {
        Bucket: dbNameUsersWasabi,
        Key: data.codeUser,
      };

      s3.getObject(params, (err, dataR) => {
        if (err) {
          // console.error('Error getting object:', err);
          console.log(`Data doesn't exist !`)
        } else {
          const dataUser = JSON.parse(dataR.Body.toString());
          const contactColl = dataUser.contactCollections;
          const editContactColl = contactColl.map(dataContact => {
            if (dataContact.roomPath === data.roomPath) {
              dataContact.name = data.valueInput
            }
            return dataContact
          });
          //   update data user
          dataUser.contactCollections = editContactColl
          const updatedParams = {
            Bucket: dbNameUsersWasabi,
            Key: data.codeUser,
            Body: JSON.stringify(dataUser),
            ContentType: 'application/json',
          };
          s3.putObject(updatedParams, (err, dataY) => {
            if (err) {
              console.error('Error updating object:', err);
            } else {
              console.log('successfuly updated data:', dataY);
            }
          });
        }
      });

    } catch (error) {
      console.log(error)
    }
  });

  // delete contact -y 
  socket.on('delete_contact', async (data) => {
    try {
      const params = {
        Bucket: dbNameUsersWasabi,
        Key: data.codeUserApp,
      };

      s3.getObject(params, (err, dataR) => {
        if (err) {
          // console.error('Error getting object:', err);
          console.log(`Data doesn't exist !`)
        } else {
          const dataUser = JSON.parse(dataR.Body.toString());
          const contactColl = dataUser.contactCollections;
          const deleteCntkColl = dataUser.deleteContactCollections;
          const filterContactColl = contactColl.filter(contact => {
            return contact.code !== data.codeContact;
          });
          const getDataContactDelete = contactColl.find(contact => {
            return contact.code === data.codeContact
          });
          //   get data user contact
          const params = {
            Bucket: dbNameUsersWasabi,
            Key: data.codeContact,
          };
          s3.getObject(params, (err, dataK) => {
            if (err) {
              // console.error('Error getting object:', err);
              console.log(`Data doesn't exist !`)
            } else {
              const dataContact = JSON.parse(dataK.Body.toString());
              const objDltContact = {
                name: getDataContactDelete.name,
                codeContact: getDataContactDelete.codeShare,
                roomPath: getDataContactDelete.roomPath,
                codeRealContact: getDataContactDelete.code,
                imgProfCntk:dataContact.imgProfileUser,
                idImgProfile: dataContact.idImgProf,
                mimeImgProf: dataContact.mimeImgProf,
                country: dataContact.country,
                birthYear: dataContact.birthYear,
                gender: dataContact.genderUser,
              }
              // //         // ceck already contact in delete collections    
              const ceckContact = deleteCntkColl.filter(dataContact => {
                return dataContact.roomPath !== getDataContactDelete.roomPath
                // return dataContact.code === data.codeContact;
              });
              if (ceckContact.length == 0) {
                deleteCntkColl.push(objDltContact);
                // let newArr = [];
                // let newArr1 = newArr.concat(deleteCntkColl);
                // let newArr2 = newArr1.push(objDltContact);

                // update data user
                dataUser.deleteContactCollections = deleteCntkColl;
                dataUser.contactCollections = filterContactColl
                const updatedParams = {
                  Bucket: dbNameUsersWasabi,
                  Key: data.codeUserApp,
                  Body: JSON.stringify(dataUser),
                  ContentType: 'application/json',
                };

                s3.putObject(updatedParams, (err, dataY) => {
                  if (err) {
                    console.error('Error updating object:', err);
                  } else {
                    console.log('successfuly updated data user:', dataY);
                  }
                });
              } else {
                // change old contact in delete contact with new delete contact
                const filterContactDlt = deleteCntkColl.filter(dataContact => {
                  return dataContact.roomPath !== getDataContactDelete.roomPath
                });
                filterContactDlt.push(objDltContact);
                // update data user
                dataUser.deleteContactCollections = filterContactDlt;
                dataUser.contactCollections = filterContactColl
                const updatedParams = {
                  Bucket: dbNameUsersWasabi,
                  Key: data.codeUserApp,
                  Body: JSON.stringify(dataUser),
                  ContentType: 'application/json',
                };

                s3.putObject(updatedParams, (err, dataY) => {
                  if (err) {
                    console.error('Error updating object:', err);
                  } else {
                    console.log('successfuly updated data user:', dataY);
                  }
                });
              }
              socket.emit('response_delete_contact', { nameContact: objDltContact.name, codeShareContact: objDltContact.codeContact })
            }
          });
        }

      });

    } catch (error) {
      console.log(error)
    }
  });

  // -y
  socket.on('dlt_coll', async data => {
    try {
      if (data) {
        const params = {
          Bucket: dbNameUsersWasabi,
          Key: data,
        };

        s3.getObject(params, async (err, dataR) => {
          if (err) {
            // console.error('Error getting object:', err);
            console.log(`Data doesn't exist !`)
          } else {
            const dataUser = JSON.parse(dataR.Body.toString());
            const dltColl = dataUser.deleteContactCollections;
            const arr = await Promise.all(dltColl.map(async contact => {
              return new Promise(async (resolve, reject) => {
                try {
                  if (contact.idImgProfile !== '') {
                    const param = {
                      Bucket: dbNameFileWasabi,
                      Key: contact.idImgProfile
                    };

                    s3.getObject(param, async (err, dataR) => {
                      if (err) {
                        reject(err);
                        return;
                      }
                      const dataFile = JSON.parse(dataR.Body.toString());
                      contact.imgProfCntk = dataFile;
                      resolve(contact);
                    });
                  } else {
                    const param = {
                      Bucket: dbNameUsersWasabi,
                      Key: contact.codeR
                    };

                    s3.getObject(param, async (err, dataR) => {
                      if (err) {
                        reject(err);
                        return;
                      }
                      const dataUser = JSON.parse(dataR.Body.toString());
                      contact.imgProfCntk = dataUser.imgProfileUser;
                      resolve(contact);
                    });
                  }
                } catch (error) {
                  reject(error);
                }
              });
            }));
            //   console.log('Object data:', jsonData);
            socket.emit('recive_dlt_coll', arr)
            // console.log(data)
          }

        });
      }
    } catch (error) {
      console.log(error)
    }
  });
  // -y
  socket.on('get_dlt_data_contact', async (data) => {
    try {
      if (data) {

        const params = {
          Bucket: dbNameUsersWasabi,
          Key: data.codeUserApp,
        };

        s3.getObject(params, async (err, dataR) => {
          if (err) {
            // console.error('Error getting object:', err);
            console.log(`Data doesn't exist !`)
          } else {
            const dataUser = JSON.parse(dataR.Body.toString());
            const dltColl = dataUser.deleteContactCollections;
            const dataArrRes = [];

            //   filter dlt contact if no apply the category methode
            if (data.Gender !== '' && data.BirthYear !== '' && data.CountryName !== '') {
              const filter = dltColl.filter(dataCntc => {
                return dataCntc.gender === data.Gender && dataCntc.birthYear === data.BirthYear && dataCntc.country === data.CountryName
              });
              // dataArrRes(filter)
              dataArrRes.concat(filter)
            }

            // // just for gender
            if (data.Gender !== '' && data.BirthYear === '' && data.CountryName === '') {
              const filterGnder = dltColl.filter(dataCntc => {
                return dataCntc.gender === data.Gender
              });
              dataArrRes.push(filterGnder)
              // dataArrRes(filterGnder)
            }

            // // just for year
            if (data.BirthYear !== '' && data.Gender === '' && data.CountryName === '') {
              const filterYear = dltColl.filter(dataCntc => {
                return `${dataCntc.birthYear}` == data.BirthYear
              });
              // dataArrRes(filterYear)
              dataArrRes.concat(filterYear)
            }

            // // just for country
            if (data.CountryName !== '' && data.Gender === '' && data.BirthYear === '') {
              const filterCountry = dltColl.filter(dataCntc => {
                return dataCntc.country === data.CountryName
              });
              // dataArrRes(filterCountry)
              dataArrRes.concat(filterCountry);
            }

            // // gender x Byear
            if (data.Gender !== '' && data.BirthYear !== '' && data.CountryName === '') {
              const filter = dltColl.filter(dataCntc => {
                return dataCntc.gender === Gender && `${dataCntc.birthYear}` === BirthYear
              });
              // dataArrRes(filter);
              dataArrRes.concat(filter)
            }

            // // Byear x country
            if (data.BirthYear !== '' && data.Gender === '' && data.CountryName !== '') {
              const filter = dltColl.filter(dataCntc => {
                return dataCntc.birthYear === data.BirthYear && dataCntc.country === data.CountryName
              });
              // dataArrRes(filter)
              dataArrRes.concat(filter);
            }

            // // gender x country
            if (data.Gender !== '' && data.BirthYear === '' && data.CountryName !== '') {
              const filter = dltColl.filter(dataCntc => {
                return dataCntc.country === data.CountryName && dataCntc.gender === data.Gender
              });
              dataArrRes(filter)
            }
            if (dltColl.length != 0) {
              const arr = await Promise.all(dataArrRes.map(async contact => {
                return new Promise(async (resolve, reject) => {
                  try {
                    if (contact.idImgProfile !== '') {
                      const param = {
                        Bucket: dbNameFileWasabi,
                        Key: contact.idImgProfile
                      };

                      s3.getObject(param, async (err, dataR) => {
                        if (err) {
                          reject(err);
                          return;
                        }
                        const dataFile = JSON.parse(dataR.Body.toString());
                        contact.imgProfCntk = dataFile;
                        resolve(contact);
                      });
                    } else {
                      const param = {
                        Bucket: dbNameUsersWasabi,
                        Key: contact.codeR
                      };

                      s3.getObject(param, async (err, dataR) => {
                        if (err) {
                          reject(err);
                          return;
                        }

                        const dataUser = JSON.parse(dataR.Body.toString());
                        contact.imgProfCntk = dataUser.imgProfileUser;
                        resolve(contact);
                      });
                    }
                  } catch (error) {
                    reject(error);
                  }
                });
              }));
              socket.emit('recive_data_Cntc_dlt', arr);
            }


            // console.log('Object data:', dataUser);

          }

        });

      }
    } catch (error) {
      console.log(error)
    }
  })

  // restore contact -y
  socket.on('restore_contact', async (data) => {
    try {
      const params = {
        Bucket: dbNameUsersWasabi,
        Key: data.codeUser,
      };

      s3.getObject(params, (err, dataR) => {
        if (err) {
          // console.error('Error getting object:', err);
          console.log(`Data doesn't exist !`)
        } else {
          const dataUser = JSON.parse(dataR.Body.toString());
          const contactColl = dataUser.contactCollections
          const deleteCntkColl = dataUser.deleteContactCollections;
          //   get data contact from contact coll
          const getDataCntc = deleteCntkColl.find(dtCntc => {
            return dtCntc.codeContact === data.codeContact
          });
          //   ceck contact for user
          const ceckContact = contactColl.find(dataCntc => {
            return dataCntc.code === getDataCntc.codeRealContact
          });
          // get data contact 
          const params = {
            Bucket: dbNameUsersWasabi,
            Key: getDataCntc.codeRealContact,
          };
          s3.getObject(params, (err, dataB) => {
            if (err) {
              // console.error('Error getting object:', err);
              console.log(`Data doesn't exist !`)
            } else {
              const dataContact = JSON.parse(dataB.Body.toString());
              const getRoomPath = getDataCntc.roomPath;
              // get data room
              const paramsRoom = {
                Bucket: dbNameRoomWasabi,
                Key: getRoomPath,
              };
              s3.getObject(paramsRoom, (err, dataK) => {
                if (err) {
                  console.log(`Data doesn't exist !`);
                  // // get img prof contact
                  const objDataContact = {
                    roomPath: getRoomPath,
                    code: dataContact.code,
                    codeShare: dataContact.codeShare,
                    name: getDataCntc.name,
                    imgProfCntk:dataContact.imgProfileUser,
                    idImgProfile: dataContact.idImgProf,
                    mimeImgProf: dataContact.mimeImgProf,
                    msgStates: 0,
                    tokenDevice: dataContact.tokenDevice
                  }
                  // push to contact coll user
                  contactColl.push(objDataContact);
                  // // remove contact on delete contact colections
                  const filterDltContactColl = deleteCntkColl.filter(dltCntk => {
                    return dltCntk.name !== getDataCntc.name;
                  });
                  if (ceckContact != undefined) {
                    socket.emit('announcement_alert', {
                      msg: `Contact with code : ${getDataCntc.code} already exist,
                      her name : ( ${getDataCntc.name} );
                    `});
                  } else {
                    dataUser.contactCollections = contactColl;
                    dataUser.deleteContactCollections = filterDltContactColl
                    // update data user
                    const updatedParams = {
                      Bucket: dbNameUsersWasabi,
                      Key: data.codeUser,
                      Body: JSON.stringify(dataUser),
                      ContentType: 'application/json',
                    };
                    s3.putObject(updatedParams, (err, dataZ) => {
                      if (err) {
                        console.error('Error updating object:', err);
                      } else {
                        console.log('successfuly updated data user:', dataZ);
                      }
                    });
                    socket.emit('update_contact_coll', contactColl);
                  }
                } else {
                  const dataRoom = JSON.parse(dataK.Body.toString());
                  // if (dataRoom !== null || dataRoom != undefined) {
                  //   const msgColl = dataRoom.message;
                  //   const filterMsg = msgColl.filter(msg => {
                  //     return msg.state === ''
                  //   })
                  //   const objDataContact = {
                  //     roomPath: getRoomPath,
                  //     code: dataContact.code,
                  //     codeShare: dataContact.codeShare,
                  //     name: getDataCntc.name,
                  //     imgProfCntk: '',
                  //     idImgProfile: dataContact.idImgProf,
                  //     mimeImgProf: dataContact.mimeImgProf,
                  //     msgStates: filterMsg.length,
                  //     tokenDevice: dataContact.tokenDevice
                  //   }
                  //   // push to contact coll user
                  //   contactColl.push(objDataContact);
                  // } else {
                  //   // // // get img prof contact
                  //   // const objDataContact = {
                  //   //   roomPath: getRoomPath,
                  //   //   code: dataContact.code,
                  //   //   codeShare: dataContact.codeShare,
                  //   //   name: getDataCntc.name,
                  //   //   // imgProfCntk:dataContact.imgProfileUser,
                  //   //   imgProfCntk: '',
                  //   //   idImgProfile: dataContact.idImgProf,
                  //   //   mimeImgProf: dataContact.mimeImgProf,
                  //   //   msgStates: 0,
                  //   //   tokenDevice: dataContact.tokenDevice
                  //   // }
                  //   // // push to contact coll user
                  //   // contactColl.push(objDataContact);
                  // }

                  // // remove contact on delete contact colections
                  
                    const msgColl = dataRoom.message;
                    const filterMsg = msgColl.filter(msg => {
                      return msg.state != true;
                    });
                    const objDataContact = {
                      roomPath: getRoomPath,
                      code: dataContact.code,
                      codeShare: dataContact.codeShare,
                      name: getDataCntc.name,
                      imgProfCntk: dataContact.imgProfCntk,
                      idImgProfile: dataContact.idImgProf,
                      mimeImgProf: dataContact.mimeImgProf,
                      msgStates: filterMsg.length,
                      tokenDevice: dataContact.tokenDevice
                    }
                    // push to contact coll user
                    contactColl.push(objDataContact);
                  const filterDltContactColl = deleteCntkColl.filter(dltCntk => {
                    return dltCntk.name !== getDataCntc.name;
                  });
                  if (ceckContact != undefined) {
                    socket.emit('announcement_alert', {
                      msg: `Contact with code : ${getDataCntc.code} already exist,
                      her name : ( ${getDataCntc.name} );
                    `});
                  } else {
                    // update data user
                    dataUser.contactCollections = contactColl;
                    dataUser.deleteContactCollections = filterDltContactColl
                    const updatedParams = {
                      Bucket: dbNameUsersWasabi,
                      Key: data.codeUser,
                      Body: JSON.stringify(dataUser),
                      ContentType: 'application/json',
                    };
                    s3.putObject(updatedParams, (err, dataZ) => {
                      if (err) {
                        console.error('Error updating object:', err);
                      } else {
                        console.log('successfuly restored contact:', dataZ);
                      }
                    });
                    socket.emit('update_contact_coll', contactColl);
                  }
                }
              });
            }
          });
        }
      });

    } catch (error) {
      console.log(error)
    }
  });

  // create group -y
  socket.on('create_group', async (data) => {
    try {
      const getObjectData = async (key) => {
        const params = {
          Bucket: dbNameUsersWasabi,
          Key: key,
        };
        try {
          const data = await s3.getObject(params).promise();
          return JSON.parse(data.Body.toString());
        } catch (error) {
          console.error(`Error getting object '${key}':`, error);
          throw error;
        }
      };
      const getAllDataFromBucket = async () => {
        const params = {
          Bucket: dbNameUsersWasabi,
        };
        try {
          const listObjectsResponse = await s3.listObjectsV2(params).promise();
          const objects = listObjectsResponse.Contents;
          const allData = await Promise.all(
            objects.map(async (object) => {
              const key = object.Key;
              const data = await getObjectData(key);
              return { key: key, data };
            })
          );
          const mapAllData = allData.map(dtbs => {
            const dataM = dtbs.data
            return dataM
          });
          //   get data user
          const dataUser = mapAllData.find(dataU => {
            // console.log(`code user : ${data.codeUserApp}, code get user ${dataU.code}`)
            // if (dataU.code === data.codeUserApp) {
            //   return dataU
            // }
            return dataU.code === data.codeUserApp
          });
          const groupCollections = dataUser.groupCollections;
          const findDblNameGroup = groupCollections.find(dataG => {
            return dataG.groupName === data.objGroup.groupName
          });
          if (findDblNameGroup == undefined) {
            const randomUUID = uuid.v4();
            const randomUUIDFIle = uuid.v4()
            // add self to be member
            const memberData = {
              callName: data.objGroup.adminName,
              code: dataUser.codeShare,
              codeR: dataUser.code,
              tokenDevice: dataUser.tokenDevice,
            }
            // add file
            let keyFile = `group-prof-${randomUUIDFIle}`
            const params = {
              Bucket: dbNameFileWasabi,
              Key: keyFile,
              Body: Buffer.from(data.base64data, 'base64'),
              ACL: 'public-read', // Opsional, untuk memberikan akses publik
            };
            s3.upload(params, (err, dataZ) => {
              if (err) {
                console.error('Error uploading file:', err);
              } else {
                console.log('File uploaded successfully:', dataZ.Location);
                const expirationTime = 31536000;
                const signedUrl = s3.getSignedUrl('getObject', {
                  Bucket: dbNameFileWasabi,
                  Key: keyFile,
                  Expires: expirationTime,
                });
                data.objGroup.groupMember.push(memberData);
                data.objGroup.groupImg = signedUrl;
                data.objGroup.idGroupImg = keyFile;
                data.objGroup.room = randomUUID;
                groupCollections.push(data.objGroup);
                dataUser.groupCollections = groupCollections;
                // update user
                const updatedParams = {
                  Bucket: dbNameUsersWasabi,
                  Key: data.codeUserApp,
                  Body: JSON.stringify(dataUser),
                  ContentType: 'application/json',
                };
                s3.putObject(updatedParams, (err, dataB) => {
                  if (err) {
                    console.error('Error updating object:', err);
                  } else {
                    console.log('successfuly updated datauser:', dataB);
                  }
                });
                 // add group to database
                const paramsGroup = {
                  Bucket: dbNameRoomWasabi,
                  Key: randomUUID,
                  Body: JSON.stringify(data.objGroup), // Mengonversi objek JSON menjadi string JSON
                  ContentType: 'application/json', // Tentukan jenis konten sebagai JSON
                };
                s3.putObject(paramsGroup, (err, dataK) => {
                  if (err) {
                    console.error('Error adding JSON object:', err);
                  } else {
                    console.log('group added successfully:', dataK);
                  }
                });
              }
            });
            socket.emit('update_group_coll', groupCollections);
            socket.emit('response_create_group', { state: true, groupName: data.objGroup.groupName })

          } else {
            socket.emit('update_group_coll', groupCollections);
            socket.emit('response_create_group', { state: true, groupName: data.objGroup.groupName })
          }

        } catch (error) {
          console.error('Error listing objects:', error);
        }
      };
      getAllDataFromBucket();

    } catch (error) {
      console.log(error)
    }
  });

  // get full data Group / room -y
  socket.on('getFull_data_group', async (data) => {
    try {
      if (data) {

        const params = {
          Bucket: dbNameRoomWasabi,
          Key: data,
        };
        s3.getObject(params, (err, dataR) => {
          if (err) {
            // console.error('Error getting object:', err);
            console.log(`Data doesn't exist !`)
          } else {
            const dataGroup = JSON.parse(dataR.Body.toString());
            socket.emit('recive_fullData_group', dataGroup);
            // console.log(data)
          }
        });
      }
    } catch (error) {
      console.log(error)
    }
  });

  // get full data room / group -y
  socket.on('getFull_data_room', async data => {
    try {
      // console.log('ok :', data)
      if (data) {
        const params = {
          Bucket: dbNameRoomWasabi,
          Key: data,
        };

        s3.getObject(params, async (err, dataR) => {
          if (err) {
            console.error('Error getting object:', err);
            // console.log(`Data doesn't exist !`)
          } else {
            const dataRoom = await JSON.parse(dataR.Body.toString());
            //  console.log(dataRoom)
            const messageColl = dataRoom.message;
            const cD = new Date().getTime();
            // console.log('data object', dataR.ContentLength);
            const filterMsgColl = messageColl.filter(msg => {
              // return msg.times.timeDeadline > cD
              return msg
            })
            dataRoom.message = filterMsgColl;
            // socket.emit('recive_msgs', {msgs:filterMsgColl});
            // console.log(dataRoom);
            // // update groups
            const updatedParams = {
              Bucket: dbNameRoomWasabi,
              Key: data,
              Body: JSON.stringify(dataRoom),
              ContentType: 'application/json',
            };

            s3.putObject(updatedParams, (err, dataX) => {
              if (err) {
                console.error('Error updating object:', err);
              } else {
                console.log('successfuly updated data room:', dataX);
              }
            });
            socket.emit('recive_fullData_room', dataRoom);

          }
        });

      }
    } catch (error) {
      console.log(error)
    }
  });

  // change call name -y
  socket.on('change_callname_group', async (data) => {
    try {
      const params = {
        Bucket: dbNameUsersWasabi,
        Key: data.codeUser,
      };
      s3.getObject(params, (err, dataR) => {
        if (err) {
          // console.error('Error getting object:', err);
          console.log(`Data doesn't exist !`)
        } else {
          const dataUser = JSON.parse(dataR.Body.toString());
          const codeShareUser = dataUser.codeShare;
          // get data group  
          const params = {
            Bucket: dbNameRoomWasabi,
            Key: data.roomPath,
          };
          s3.getObject(params, (err, dataY) => {
            if (err) {
              // console.error('Error getting object:', err);
              console.log(`Data doesn't exist !`)
            } else {
              const dataGroup = JSON.parse(dataY.Body.toString());
              const groupMember = dataGroup.groupMember;

              // // update for database group
              const newGroupMember = groupMember.map(dataMember => {
                if (dataMember.code === codeShareUser) {
                  dataMember.callName = data.callName
                }
                return dataMember
              });

              if (data.callName !== '') {
                if (dataUser.codeShare === dataGroup.adminCode) {
                  console.log('you are admin !')
                  dataGroup.adminName = data.callName;
                  // update data group 1
                  const updatedParams = {
                    Bucket: dbNameRoomWasabi,
                    Key: data.roomPath,
                    Body: JSON.stringify(dataGroup),
                    ContentType: 'application/json',
                  };

                  s3.putObject(updatedParams, (err, dataK) => {
                    if (err) {
                      console.error('Error updating object:', err);
                    } else {
                      console.log('successfuly updated data group:', dataK);
                    }
                  });
                } else {
                  console.log('you are not admin !')
                }
                // update data group 2
                dataGroup.groupMember = newGroupMember;
                const updatedParams = {
                  Bucket: dbNameRoomWasabi,
                  Key: data.roomPath,
                  Body: JSON.stringify(dataGroup),
                  ContentType: 'application/json',
                };

                s3.putObject(updatedParams, (err, dataK) => {
                  if (err) {
                    console.error('Error updating object:', err);
                  } else {
                    console.log('successfuly updated data group:', dataK);
                  }
                });
              }

            }

          });
        }

      });

    } catch (error) {
      console.log(error)
    }
  });

  // join group -y 
  socket.on('join_group', async (data) => {
    try {
      const params = {
        Bucket: dbNameRoomWasabi,
        Key: data.roomPath,
      };

      s3.getObject(params, (err, dataR) => {
        if (err) {
          // console.error('Error getting object:', err);
          console.log(`Data doesn't exist !`);
          socket.emit('group_state', { msg: `Group with code room ${data.roomPath} doesn't exist.`, state: false });
        } else {
          const dataGroup = JSON.parse(dataR.Body.toString());
          const memberColl = dataGroup.groupMember;
          //   get data user
          const params = {
            Bucket: dbNameUsersWasabi,
            Key: data.codeUser,
          };
          s3.getObject(params, (err, dataY) => {
            if (err) {
              console.log(`Data doesn't exist !`)
            } else {
              const dataUser = JSON.parse(dataY.Body.toString());
              const groupColl = dataUser.groupCollections;
              const msgStatuses = dataUser.msgStatuses
              const objDataMember = {
                callName: data.callName,
                code: dataUser.codeShare,
                codeR: dataUser.code,
                tokenDevice: dataUser.tokenDevice
              }
              // check member
              const findMember = memberColl.find(member => {
                return member.codeR === dataUser.code;
              });
              if (findMember == undefined) {
                memberColl.push(objDataMember);
                dataGroup.groupMember = memberColl
                // update group
                const updatedParams = {
                  Bucket: dbNameRoomWasabi,
                  Key: data.roomPath,
                  Body: JSON.stringify(dataGroup),
                  ContentType: 'application/json',
                };
                s3.putObject(updatedParams, (err, dataR) => {
                  if (err) {
                    console.error('Error updating object:', err);
                  } else {
                    console.log('successfuly updated data group:', dataR);
                  }
                });
                const msgGroup = dataGroup.message;
                const filtermsg = msgGroup.filter(msg => {
                  return msg.code !== dataUser.code && msg.state == false
                });
                // update data user
                groupColl.push(dataGroup);
                dataUser.msgStatuses = msgStatuses + filtermsg.length;
                dataUser.groupCollections = groupColl;
                const updatedParamsUser = {
                  Bucket: dbNameUsersWasabi,
                  Key: data.codeUser,
                  Body: JSON.stringify(dataUser),
                  ContentType: 'application/json',
                };
                s3.putObject(updatedParamsUser, (err, dataX) => {
                  if (err) {
                    console.error('Error updating object:', err);
                  } else {
                    console.log('successfuly updated data user:', dataX);
                  }
                });
                socket.emit('group_state', { msg: `You have joined this group, with name : ${dataGroup.groupName}`, state: true });
              } else {
                socket.emit('group_state', { msg: `you have already join to this group`, state: false });
              }
            }
          });
        }
      });
    } catch (error) {
      console.log(error)
    }
  });

  // edit group -y
  socket.on('edit_group', (data) => {
    try {
      const params = {
        Bucket: dbNameRoomWasabi,
        Key: data.roomPath,
      };
      s3.getObject(params, (err, dataR) => {
        if (err) {
          console.log(`Data doesn't exist !`)
        } else {
          const dataGroup = JSON.parse(dataR.Body.toString());
          const groupMember = dataGroup.groupMember;
          // for change admin check
          const findMember = groupMember.find(member => {
            return member.code === data.newAdminCode
          });
          if (data.base64data !== '') {
            const randomUUIDFIle = uuid.v4();
            // remove file
            const paramsF = {
              Bucket: dbNameFileWasabi,
              Key: dataGroup.idGroupImg,
            };
            s3.deleteObject(paramsF, (err, data) => {
              if (err) {
                console.error('Error deleting object:', err);
              } else {
                console.log('file deleted successfully:', data);
              }
            });
            // add file
            let keyFile = `img-prof-group-${randomUUIDFIle}`;
            const params = {
              Bucket: dbNameFileWasabi,
              Key: keyFile,
              Body: Buffer.from(data.base64data, 'base64'),
              ACL: 'public-read', // Opsional, untuk memberikan akses publik
            };
            s3.upload(params, (err, data) => {
              if (err) {
                console.error('Error uploading file:', err);
              } else {
                console.log('File uploaded successfully:', data.Location);
                const expirationTime = 31536000;
                const signedUrl = s3.getSignedUrl('getObject', {
                  Bucket: dbNameFileWasabi,
                  Key: keyFile,
                  Expires: expirationTime,
                });
                dataGroup.idGroupImg = keyFile;
                dataGroup.groupImg = signedUrl;
              }
            });
          }
          dataGroup.groupName = data.groupName;
          if (findMember) {
            dataGroup.adminCode = findMember.code;
            dataGroup.adminName = findMember.callName;
          } else {
            socket.emit('response_change_group', { msg: `There are no members with code ${data.newAdminCode} in this group`, state: false })
          }
          // update each member
          const updateEachMember = groupMember.forEach(member => {
            // update datauser
            const params = {
              Bucket: dbNameUsersWasabi,
              Key: member.codeR,
            };
            s3.getObject(params, (err, dataG) => {
              if (err) {
                console.error('Error getting data member:', err);
              } else {
                const existingData = JSON.parse(dataG.Body.toString());
                const groupColl = existingData.groupCollections
                const filterGroup = groupColl.filter(group => {
                  return group.room !== data.roomPath
                });
                filterGroup.push(dataGroup)
                existingData.groupCollections = filterGroup
                const updatedParams = {
                  Bucket: dbNameUsersWasabi,
                  Key: member.codeR,
                  Body: JSON.stringify(existingData),
                  ContentType: 'application/json',
                };
                s3.putObject(updatedParams, (err, data) => {
                  if (err) {
                    console.error('Error updating object:', err);
                  } else {
                    console.log('successfuly updated data member:', data);
                  }
                });

              }
            })
            // return member
          });

          // update group
          const updatedParams = {
            Bucket: dbNameRoomWasabi,
            Key: dataGroup.room,
            Body: JSON.stringify(dataGroup),
            ContentType: 'application/json',
          };

          s3.putObject(updatedParams, (err, dataB) => {
            if (err) {
              console.error('Error updating object:', err);
            } else {
              console.log('successfuly updated group:', dataB);
              socket.emit('response_change_group', { msg: `Group has been update 👌`, state: true })
            }
          });

        }
      });



    } catch (error) {
      console.log(error)
    }
  });

  // change call name -y
  socket.on('change_call_name', async (data) => {
    try {
      const params = {
        Bucket: dbNameRoomWasabi,
        Key: data.room,
      };

      s3.getObject(params, (err, dataR) => {
        if (err) {
          // console.error('Error getting object:', err);
          console.log(`Data doesn't exist !`)
        } else {
          const dataGroup = JSON.parse(dataR.Body.toString());
          const getMember = dataGroup.groupMember;
          const mapMember = getMember.map(member => {
            if (member.code === data.codeShare) {
              member.callName = data.inputName
            }
            return member
          });
          dataGroup.groupMember = mapMember
          if (dataGroup.adminCode === data.codeShare) {
            dataGroup.adminName = data.inputName
          }
          // // update data group
          const updatedParams = {
            Bucket: dbNameRoomWasabi,
            Key: data.room,
            Body: JSON.stringify(dataGroup),
            ContentType: 'application/json',
          };
          s3.putObject(updatedParams, (err, dataZ) => {
            if (err) {
              console.error('Error updating object:', err);
            } else {
              console.log('successfuly updated data group !');
              socket.emit('response_change_group', { msg: `Group has been updated !!`, state: true });
            }
          });
        }

      });
    } catch (error) {
      console.log(error)
    }
    // console.log(getGroup);
  });

  // remove member group -y
  socket.on('remove_member_group', async (data) => {
    try {
      const params = {
        Bucket: dbNameRoomWasabi,
        Key: data.roomPath,
      };
      s3.getObject(params, (err, dataR) => {
        if (err) {
          console.log(`Data doesn't exist !`)
        } else {
          const dataGroup = JSON.parse(dataR.Body.toString());
          const groupMember = dataGroup.groupMember;
          const findMember = groupMember.find(member => {
            return member.code === data.removeCode
          });
          if (findMember != undefined && findMember.code !== dataGroup.adminCode) {
            // get data user
            const params1 = {
              Bucket: dbNameUsersWasabi,
              Key: findMember.codeR
            };
            s3.getObject(params1, (err, dataU) => {
              if(err) {
                console.log(`data doesn't exist`, err)
              } else {
                const dataUser = JSON.parse(dataU.Body.toString());
                const groupColl = dataUser.groupCollections;
                const filterGroupColl = groupColl.filter(group => {
                  return group.room !== data.roomPath
                });
                const filterMember = groupMember.filter(member => {
                  return member.code !== data.removeCode
                });
                dataGroup.groupMember = filterMember;
                dataUser.groupCollections = filterGroupColl
                 //  update group
                const updatedParams = {
                  Bucket: dbNameRoomWasabi,
                  Key: data.roomPath,
                  Body: JSON.stringify(dataGroup),
                  ContentType: 'application/json',
                };
                s3.putObject(updatedParams, (err, dataG) => {
                  if (err) {
                    console.error('Error updating object:', err);
                  } else {
                    console.log('successfuly updated data group:');
                  }
                });
                // console.log(dataUser);
                // update user
                    const updatedParamsU = {
                      Bucket: dbNameUsersWasabi,
                      Key: dataUser.code,
                      Body: JSON.stringify(dataUser),
                      ContentType: 'application/json',
                    };
                    s3.putObject(updatedParamsU, (err, dataR) => {
                      if (err) {
                        console.error('Error updating object:', err);
                      } else {
                        console.log('successfuly updated data user:', dataUser.code);
                      }
                    });
                socket.emit('removed_response', { msg: `Member with code : ${data.removeCode} has been removed !`, state: true });
              }

            })
          } else {
            if (findMember != undefined && dataGroup.adminCode === data.removeCode) {
              console.log('You are the admin !');
              socket.emit('removed_response', { msg: `You are the admin, you can't remove your self. Change with other member.`, state: false });
            } else {
              socket.emit('removed_response', { msg: `Member with code : ${data.removeCode} doesn't exist !. Change with other member.`, state: true });
            }
          }
        }
      });
    } catch (error) {
      console.log(error)
    }
  })

  // leave group -y
  socket.on('leave_group', async (data) => {
    try {
      const params = {
        Bucket: dbNameRoomWasabi,
        Key: data.roomPath,
      };
      s3.getObject(params, (err, dataR) => {
        if (err) {
          console.log(`Data doesn't exist !`,err)
        } else {
          const dataGroup = JSON.parse(dataR.Body.toString());
          const groupMember = dataGroup.groupMember;
          const maping = groupMember.map(member => {
            if (data.codeUserShare === member.code) {
              // get data member
              const params = {
                Bucket: dbNameUsersWasabi,
                Key: member.codeR,
              };
              s3.getObject(params, (err, dataP) => {
                if (err) {
                  console.log(`Data doesn't exist !`, err)
                } else {
                  const dataMember = JSON.parse(dataP.Body.toString());
                  const groupColl = dataMember.groupCollections;
                  const newGroupColl = groupColl.filter(group => {
                    return group.room !== data.roomPath
                  });
                  dataMember.groupCollections = newGroupColl;
                  // // //   update data user each member
                  const updatedParams = {
                    Bucket: dbNameUsersWasabi,
                    Key: dataMember.code,
                    Body: JSON.stringify(dataMember),
                    ContentType: 'application/json',
                  };
                  s3.putObject(updatedParams, (err, dataY) => {
                    if (err) {
                      console.error('Error updating object:', err);
                    } else {
                      console.log('remove group succesfully',);
                      socket.emit('update_group_coll', newGroupColl);
                      socket.emit('announcement_alert_special', {
                        msg: `You have left the group, with the name : ${dataGroup.groupName}`,
                        state: true
                      });
                    }
                  });
                }
              });
            }
          });
          const filterMember = groupMember.filter(member => {
            return member.code !== data.codeUserShare
          });
          dataGroup.groupMember = filterMember;
          //update group
          const updatedParamsG = {
            Bucket: dbNameRoomWasabi,
            Key: data.roomPath,
            Body: JSON.stringify(dataGroup),
            ContentType: 'application/json',
          };
          s3.putObject(updatedParamsG, (err, dataY) => {
            if (err) {
              console.error('Error updating object:', err);
            } else {
              console.log('member has been deleted:', dataY);
            }
          });
        }
      });
    } catch (error) {
      console.log(error)
    }
  });

  // delete group -y
  socket.on('delete_group', async (data) => {
    try {
      const params = {
        Bucket: dbNameRoomWasabi,
        Key: data.roomPath,
      };
      s3.getObject(params, (err, dataR) => {
        if (err) {
          console.log(`Data doesn't exist !`)
        } else {
          const dataGroup = JSON.parse(dataR.Body.toString());
          const groupMember = dataGroup.groupMember;
          const maping = groupMember.map(member => {
            const params = {
              Bucket: dbNameUsersWasabi,
              Key: member.codeR,
            };
            s3.getObject(params, (err, dataP) => {
              if (err) {
                console.log(`Data doesn't exist !`, err)
              } else {
                const dataMember = JSON.parse(dataP.Body.toString());
                const groupColl = dataMember.groupCollections;
                const newGroupColl = groupColl.filter(group => {
                  return group.room !== data.roomPath
                });
                dataMember.groupCollections = newGroupColl
                // //   update data user each member
                const updatedParams = {
                  Bucket: dbNameUsersWasabi,
                  Key: dataMember.code,
                  Body: JSON.stringify(dataMember),
                  ContentType: 'application/json',
                };
                s3.putObject(updatedParams, (err, dataY) => {
                  if (err) {
                    console.error('Error updating object:', err);
                  } else {
                    console.log('remove group successfully !');
                  }
                });
                socket.broadcast.emit('attention_delete_group', { msg: `Group with name ${dataGroup.groupName} has been deleted by the admin ( ${dataGroup.adminName} ) !`, groupRoom: data.roomPath, newGroupColl: newGroupColl });
              }
            });
          });
          //delete file image group
          const paramsFile = {
            Bucket: dbNameFileWasabi,
            Key: dataGroup.idGroupImg,
          };
          s3.deleteObject(paramsFile, (err, dataY) => {
            if (err) {
              console.error('Error deleting object:', err);
            } else {
              console.log('File Group successfully deleted !');
            }
          });
          //   remove data group from database
          const params = {
            Bucket: dbNameRoomWasabi,
            Key: data.roomPath,
          };
          s3.deleteObject(params, (err, dataY) => {
            if (err) {
              console.error('Error deleting object:', err);
            } else {
              console.log('Group has been removed from database !');
            }
          });
        };
      });

    } catch (error) {
      console.log(error)
    }
  });

  // join chat page -y
  socket.on('join_chat_page', async (data) => {
   try {
     // for contact
     if (data.type === 'contact') {
       socket.join(data.roomPath);
       //get data room  
       const params = {
         Bucket: dbNameRoomWasabi,
         Key: data.roomPath,
       };
       s3.getObject(params, (err, dataR) => {
         if (err) {
           // if data doesn't exist
           const objContactRoom = {
             room: data.roomPath,
             message: [],
             type: data.type,
             groupName: '',
             groupMakerCode: '',
             adminCode: '',
             adminName: '',
             groupMember: [],
             groupImg: '',
             joinedPeopleMeeting: [],
             msgStatuses: 0,
             imgGroup: '',
             mimeGroupImg: '',
             fileExtGroupImg: ''
           }
           const params = {
             Bucket: dbNameRoomWasabi,
             Key: data.roomPath,
             Body: JSON.stringify(objContactRoom), // Mengonversi objek JSON menjadi string JSON
             ContentType: 'application/json', // Tentukan jenis konten sebagai JSON
           };
           s3.putObject(params, (err, dataY) => {
             if (err) {
               console.error('Error adding JSON object:', err);
             } else {
               console.log('Room data added successfully !',);
             }

           });
         } else {
           const dataRoom = JSON.parse(dataR.Body.toString());
           const messageColl = dataRoom.message;
           // get data user`
           const paramsUser = {
             Bucket: dbNameUsersWasabi,
             Key: data.codeUserApp
           };
           s3.getObject(paramsUser, (err, dataR) => {
             if (err) {
               console.error('Error getting object:', err);
             } else {
               const dataUser = JSON.parse(dataR.Body.toString());
               const contactColl = dataUser.contactCollections;
               const stateMsgs = dataUser.msgStatuses;
               const currentDate = new Date().getTime();
              //  const filterMsgs = messageColl.filter(msg => {
              //    return msg.state == false
              //  })
               const manipulateMsgs = messageColl.map(msg => {
                 if (msg.codeShare !== data.codeUserShare) {
                   msg.state = true
                  }
                  return msg
                });
                const filterNewMsg = manipulateMsgs.filter(msg => {
                  return msg.times.timeDeadline > currentDate
                });
            // console.log(filterNewMsg.length)
               const manipulateCntcColl = contactColl.map(cntc => {
                 if (cntc.roomPath === data.roomPath) {
                   cntc.msgStates = 0
                  //  console.log(cntc)
                 }
                 return cntc
               });
               // update data room
               
               dataRoom.message = filterNewMsg
               const updatedParams = {
                 Bucket: dbNameRoomWasabi,
                 Key: data.roomPath,
                 Body: JSON.stringify(dataRoom),
                 ContentType: 'application/json',
               };
               s3.putObject(updatedParams, (err, dataX) => {
                 if (err) {
                   console.error('Error updating object:', err);
                 } else {
                   console.log('successfuly updated data room:');
                 }
               });
               // update data user
               dataUser.contactCollections = manipulateCntcColl;
               dataUser.msgStatuses = 0;
               const updatedParamsUser = {
                 Bucket: dbNameUsersWasabi,
                 Key: data.codeUserApp,
                 Body: JSON.stringify(dataUser),
                 ContentType: 'application/json',
               };
               s3.putObject(updatedParamsUser, (err, dataX) => {
                 if (err) {
                   console.error('Error updating object:', err);
                 } else {
                   console.log('successfuly updated data user:',dataRoom);
                 }
               });
             }
           });
         }
       });
     }
     // for group
     if (data.type === 'group') {
       const params = {
         Bucket: dbNameRoomWasabi,
         Key: data.roomPath,
       };
       s3.getObject(params, (err, dataR) => {
         if (err) {
           console.log(`Data doesn't exist !`, err)
         } else {
           const dataRoom = JSON.parse(dataR.Body.toString());
           const messageColl = dataRoom.message;
           const paramsUser = {
             Bucket: dbNameUsersWasabi,
             Key: data.codeUserApp
           };
           s3.getObject(paramsUser, (err, data) => {
             if (err) {
               console.error('Error getting object:', err);
             } else {
               const dataUser = JSON.parse(data.Body.toString());
               const stateMsgs = dataUser.msgStatuses;
               const groupColl = dataUser.groupCollections
               const filterMsgs = messageColl.filter(msg => {
                 return msg.state == false
               });
               const currentDate = new Date().getTime();
               const manipulateMsgs = messageColl.map(msg => {
                 if (msg.codeShare !== data.codeUserShare) {
                   msg.state = true
                 }
                 return msg
               });
               const filterNewMsg = manipulateMsgs.filter(msg => {
                  return msg.times.timeDeadline < currentDate
               });
               dataUser.groupCollections = groupColl;
               dataUser.msgStatuses = stateMsgs - filterMsgs.length;
                   // update data user
               const updatedParams = {
                 Bucket: dbNameUsersWasabi,
                 Key: dataUser.code,
                 Body: JSON.stringify(dataUser),
                 ContentType: 'application/json',
               };
               s3.putObject(updatedParams, (err, dataY) => {
                 if (err) {
                   console.error('Error updating object:', err);
                 } else {
                   console.log('successfuly updated data user:');
                 }
               });
               dataRoom.message = filterNewMsg;
               dataRoom.msgStatuses = 0;
               // update data room
               const updatedParamsRoom = {
                 Bucket: dbNameRoomWasabi,
                 Key: data.roomPath,
                 Body: JSON.stringify(dataRoom),
                 ContentType: 'application/json',
               };
               s3.putObject(updatedParamsRoom, (err, dataY) => {
                 if (err) {
                   console.error('Error updating object:', err);
                 } else {
                   console.log('successfuly updated data room:', dataY);
                 }
               });
             }
           });
         
          }
       })
     }
   } catch (error) {
     console.log(error);
   }
  })

  // save location -y
  socket.on('save_location', async (data) => {
    try {
      const params = {
        Bucket: dbNameUsersWasabi,
        Key: data.codeUserApp,
      };
      s3.getObject(params, (err, dataM) => {
        if (err) {
          console.error('Error getting object:', err);
        } else {
          const existingData = JSON.parse(dataM.Body.toString());
          // Misalnya, kita akan menambahkan properti baru atau memperbarui yang sudah ada
          if (existingData.stateLocation) {
            socket.emit('state_location_att', { msg: `Your location is active !`, state: true })
          } else {
            socket.emit('state_location_att', { msg: `Your location is off !`, state: false })
          }
          existingData.stateLocation = data.stateLocation;
          existingData.myLocation = data.myLocation;
          // console.log(existingData);
          const updatedParams = {
            Bucket: dbNameUsersWasabi,
            Key: data.codeUserApp,
            Body: JSON.stringify(existingData),
            ContentType: 'application/json',
          };

          s3.putObject(updatedParams, (err, dataZ) => {
            if (err) {
              console.error('Error updating object:', err);
            } else {
              console.log('successfuly updated location !',);
              socket.broadcast.emit('attent_state_location', { stateLocation: data.stateLocation, codeUserShare: existingData.codeShare })
            }
          });

        }
      });
    } catch (error) {
      console.log(error)
    }
  });

  // -y
  socket.on('update_my_location', (data) => {
    try {
      const params = {
        Bucket: dbNameUsersWasabi,
        Key: data.codeUser,
      };
      s3.getObject(params, (err, dataR) => {
        if (err) {
          console.error(`data user doesn't exist ! `, err);
        } else {
          const existingData = JSON.parse(dataR.Body.toString());
          // Misalnya, kita akan menambahkan properti baru atau memperbarui yang sudah ada
          existingData.myLocation = data.newLocation;
          const updatedParams = {
            Bucket: dbNameUsersWasabi,
            Key: data.codeUser,
            Body: JSON.stringify(existingData),
            ContentType: 'application/json',
          };
          s3.putObject(updatedParams, (err, dataK) => {
            if (err) {
              console.error('Error updating object:', err);
            } else {
              console.log('successfuly updated location !');
            }
          });
        }
      });
    } catch (error) {
      console.log(error)
    }
  })
  // get location -y
  socket.on('get_location', async (data) => {
    try {
      // get data user
      const params = {
        Bucket: dbNameUsersWasabi,
        Key: data.codeUserApp,
      };
      s3.getObject(params, (err, dataR) => {
        if (err) {
          console.log(`Data doesn't exist !`, err)
        } else {
          const dataUser = JSON.parse(dataR.Body.toString());
          const userLocationData = dataUser.myLocation;
          const stateLocation = dataUser.stateLocation;
          console.log('get location !')
          socket.emit('recive_location', { userLocation: userLocationData, stateLocation });
        }
      });
    } catch (error) {
      console.log(error)
    }
  });

  // get Location for group member -y
  socket.on('get_location_group', async (data) => {
    try {
      const params = {
        Bucket: dbNameRoomWasabi,
        Key: data.groupPath,
      };
      s3.getObject(params, (err, dataR) => {
        if (err) {
          console.log(`Data doesn't exist !`, err)
        } else {
          const dataGroup = JSON.parse(dataR.Body.toString());
          const groupMember = dataGroup.groupMember;
          const filterMember = groupMember.filter(member => {
            return member.code !== data.codeUserShare
          });
          // get member location exept me
          const getMemberLocation = filterMember.map(async (member) => {
            // get data user each member
            const params = {
              Bucket: dbNameUsersWasabi,
              Key: member.codeR,
            };
            s3.getObject(params, (err, dataY) => {
              if (err) {
                console.log(`Data doesn't exist !`, err)
              } else {
                const dataMember = JSON.parse(dataY.Body.toString());
                const memberLocation = dataMember.myLocation;
                const stateLocation = dataMember.stateLocation;
                if (stateLocation) {
                  const objDataMember = {
                    callName: member.callName,
                    codeShare: member.code,
                    latitude: memberLocation.latitude,
                    longitude: memberLocation.longitude,
                    latitudeDelta: 0.0922,
                    longitudeDelta: 0.0421,
                    stateLocation: stateLocation
                  }
                  socket.emit('recive_location_group', { memberData: objDataMember });
                }
              }
            });
            return member
          });
          // get member location all
          const getMemberLocationAll = groupMember.map(async (member) => {
            const params = {
              Bucket: dbNameUsersWasabi,
              Key: member.codeR,
            };
            s3.getObject(params, (err, dataY) => {
              if (err) {
                console.log(`Data doesn't exist !`, err)
              } else {
                const dataMember = JSON.parse(dataY.Body.toString());
                const memberLocation = dataMember.myLocation;
                const stateLocation = dataMember.stateLocation;
                const objDataMember = {
                  callName: member.callName,
                  codeShare: member.code,
                  codeR: member.codeR,
                  latitude: memberLocation.latitude,
                  longitude: memberLocation.longitude,
                  latitudeDelta: 0.0922,
                  longitudeDelta: 0.0421,
                  stateLocation: stateLocation
                }
                if (dataMember.stateLocation == true) {
                  socket.emit('recive_location_group_all', { memberData: objDataMember });
                }
                socket.emit('recive_location_group_all_coll', { memberDataColl: objDataMember });
              }
            });
            // return objDataMember
            return member
          });
        }
      });

    } catch (error) {
      console.log(error)
    }
  });

  socket.on('push_adsImg_to_dbs_ads', async (data) => {
    try {
      const randomUUID = uuid.v4()
      const randomUUIDFIle = uuid.v4();
      let keyAd = `ads-data-${randomUUID}`;
      let keyFile =`ads-img-${randomUUIDFIle}`;
      const paramsFile = {
        Bucket: dbNameFileWasabi,
        Key: keyFile,
        Body: Buffer.from(data.dataAds.base64Img, 'base64'),
        ACL: 'public-read', // Opsional, untuk memberikan akses publik
      };
      s3.upload(paramsFile, (err, dataR) => {
        if (err) {
          console.error('Error uploading file:', err);
        } else {
            // console.log('File uploaded successfully:', dataR.Location);
            const expirationTime = 2678400;
            const signedUrl = s3.getSignedUrl('getObject', {
              Bucket: dbNameFileWasabi,
              Key: keyFile,
              Expires: expirationTime,
            });
            console.log(signedUrl)
            data.dataAds.idAd = keyAd;
            data.dataAds.idFile = keyFile;
            data.dataAds.base64Img = signedUrl;
            if(data.dataAds.type === 'video') {
              const randomUUIDFIleVideo = uuid.v4();
              let keyFileV = `ads-video-${randomUUIDFIleVideo}`
              const paramsFileV = {
                Bucket: dbNameFileWasabi,
                Key: keyFileV,
                Body: Buffer.from(data.dataAds.base64Video, 'base64'),
                ACL: 'public-read', // Opsional, untuk memberikan akses publik
              };
              s3.upload(paramsFileV, (err, dt) => {
                if(err) {
                  console.log('error upload poster img', err)
                } else {
                  const expirationTime = 2678400;
                  const signedUrlV = s3.getSignedUrl('getObject', {
                    Bucket: dbNameFileWasabi,
                    Key: keyFileV,
                    Expires: expirationTime,
                  });
                  data.dataAds.idFileV = keyFileV;
                  data.dataAds.base64Video = signedUrlV;
                  const params = {
                    Bucket: dbNameAdsWasabi,
                    Key: keyAd,
                    Body: JSON.stringify(data.dataAds), // Mengonversi objek JSON menjadi string JSON
                    ContentType: 'application/json', // Tentukan jenis konten sebagai JSON
                  };
                  s3.putObject(params, (err, dataR) => {
                    if (err) {
                      console.error('Error adding JSON object:', err);
                    } else {
                      console.log('successfully added ad:',signedUrlV);
                    }
            
                  });
                }
              })

            }
          // }
          // addad
          if(data.dataAds.type === 'image') {
            const params = {
              Bucket: dbNameAdsWasabi,
              Key: keyAd,
              Body: JSON.stringify(data.dataAds), // Mengonversi objek JSON menjadi string JSON
              ContentType: 'application/json', // Tentukan jenis konten sebagai JSON
            };
            s3.putObject(params, (err, dataR) => {
              if (err) {
                console.error('Error adding JSON object:', err);
              } else {
                console.log('successfully added ad:', dataR);
                console.log(params);
              }
      
            });
          }
        }
      });
    } catch (error) {
      console.log(error)
    }
  });

  // start advertising -y
  socket.on('set_start_advertising', async (data) => {
    try {
      // time now
      const date = new Date();
      const yearNow = date.getFullYear();
      const monthNow = date.getMonth();
      const dayNow = date.getDate();
      // time ends
      const timeEnds = new Date(2023, monthNow, dayNow + 1);
      const d = timeEnds.getDate();
      const c = timeEnds.getMonth();
      const e = timeEnds.getFullYear();
      const timeNowObj = {
        timeNormal: `${dayNow}-${monthNow + 1}-${yearNow}`,
        timeMili: date.getTime()
      }
      const timeEndsObj = {
        timeNormal: `${d}-${c + 1}-${e}`,
        timeMili: timeEnds.getTime()
      }
      // get data ads
      const getObjectData = async (key) => {
        const params = {
          Bucket: dbNameAdsWasabi,
          Key: key,
        };
        try {
          const data = await s3.getObject(params).promise();
          return JSON.parse(data.Body.toString());
        } catch (error) {
          console.error(`Error getting object '${key}':`, error);
          throw error;
        }
      };
      const getAllDataFromBucket = async () => {
        const params = {
          Bucket: dbNameAdsWasabi,
        };
        try {
          const listObjectsResponse = await s3.listObjectsV2(params).promise();
          const objects = listObjectsResponse.Contents;
          const allData = await Promise.all(
            objects.map(async (object) => {
              const key = object.Key;
              const data = await getObjectData(key);
              return { key: key, data };
            })
          );
          const mapAllData = allData.map(dtbs => {
            const dataM = dtbs.data
            return dataM
          });
          const newAdsColl = mapAllData.map(ads => {
            if (ads.price.transactionId === data.price.transactionId) {
              ads.statePayment = true;
              ads.statePaymentText = 'settlement';
              ads.times = {
                timeStart: timeNowObj,
                timeEnds: timeEndsObj,
                timeDeadline: {
                  timeNormal: ``,
                  timeMili: 0
                }
              }
            };

            const updatedParams = {
              Bucket: dbNameAdsWasabi,
              Key: ads.idAd,
              Body: JSON.stringify(ads),
              ContentType: 'application/json',
            };

            s3.putObject(updatedParams, (err, dataK) => {
              if (err) {
                console.error('Error updating object:', err);
              } else {
                console.log('successfuly updated data ad:', dataK);
              }
            });

            return ads
          });
        } catch (error) {
          console.error('Error listing objects:', error);
        }
      };

      getAllDataFromBucket();

    } catch (error) {
      console.log(error)
    }
  });

  // update ads data -y
  socket.on('update_ads_data', async (data) => {
    const getObjectData = async (key) => {
      const params = {
        Bucket: dbNameAdsWasabi,
        Key: key,
      };
      try {
        const data = await s3.getObject(params).promise();
        return JSON.parse(data.Body.toString());
      } catch (error) {
        console.error(`Error getting object '${key}':`, error);
        throw error;
      }
    };
    const getAllDataFromBucket = async () => {
      const params = {
        Bucket: dbNameAdsWasabi,
      };
      try {
        const listObjectsResponse = await s3.listObjectsV2(params).promise();
        const objects = listObjectsResponse.Contents;
        const allData = await Promise.all(
          objects.map(async (object) => {
            const key = object.Key;
            const data = await getObjectData(key);
            return { key: key, data };
          })
        );
        const mapAllData = allData.map(dtbs => {
          const dataM = dtbs.data
          return dataM
        });
        if (mapAllData != undefined || mapAllData !== null || mapAllData.length !== 0) {
          // loop data ads for update
          const newCollAds = mapAllData.map(ad => {
            SnapApi.transaction.status(ad.price.orderId)
              .then((response) => {
                // Sample transactionStatus handling logic
                const transactionStatus = response.transaction_status;
                const fraudStatus = response.fraud_status
                if (transactionStatus == 'capture') {
                  // capture only applies to card transaction, which you need to check for the fraudStatus
                  if (fraudStatus == 'challenge') {
                    // TODO set transaction status on your databaase to 'challenge'
                    ad.statePayment = false
                    ad.statePaymentText = transactionStatus
                    ad.price.type = response.payment_type
                    if (response.payment_type === 'bank_transfer') {
                      let vaNumber = response.va_numbers;
                      ad.price.vaNumber = vaNumber.va_number;
                      ad.times.timeDeadline.timeNormal = response.expiry_time
                    }
                  } else if (fraudStatus == 'accept') {
                    ad.statePayment = true
                    ad.statePaymentText = transactionStatus
                    ad.price.type = response.payment_type
                    if (response.payment_type === 'bank_transfer') {
                      let vaNumber = response.va_numbers;
                      ad.price.vaNumber = vaNumber.va_number;
                      ad.times.timeDeadline.timeNormal = response.expiry_time
                    }
                    // TODO set transaction status on your databaase to 'success'
                  }
                } else if (transactionStatus == 'settlement') {
                  ad.statePayment = true
                  ad.statePaymentText = transactionStatus
                  ad.price.type = response.payment_type
                  if (response.payment_type === 'bank_transfer') {
                    let vaNumber = response.va_numbers;
                    ad.price.vaNumber = vaNumber.va_number;
                    ad.times.timeDeadline.timeNormal = response.expiry_time
                  }
                  // TODO set transaction status on your databaase to 'success'
                } else if (transactionStatus == 'deny') {
                  // TODO you can ignore 'deny', because most of the time it allows payment retries
                  // and later can become success
                  // delete ad
                  ad.statePayment = false
                  ad.statePaymentText = transactionStatus
                } else if (transactionStatus == 'cancel' ||
                  transactionStatus == 'expire') {
                  // TODO set transaction status on your databaase to 'failure'
                  // delete ad
                  ad.statePayment = false
                  ad.statePaymentText = transactionStatus
                } else if (transactionStatus == 'pending') {
                  // TODO set transaction status on your databaase to 'pending' / waiting payment
                  ad.statePayment = false
                  ad.statePaymentText = transactionStatus
                  ad.price.type = response.payment_type
                  if (response.payment_type === 'bank_transfer') {
                    let vaNumber = response.va_numbers;
                    ad.price.vaNumber = vaNumber.va_number;
                    ad.times.timeDeadline.timeNormal = response.expiry_time
                  }
                }
                // update all data ads
                const updatedParams = {
                  Bucket: dbNameAdsWasabi,
                  Key: ad.idAd,
                  Body: JSON.stringify(ad),
                  ContentType: 'application/json',
                };
                s3.putObject(updatedParams, (err, dataX) => {
                  if (err) {
                    console.error('Error updating object:', err);
                  } else {
                    console.log('successfuly updated data ad: !');
                  }
                });
                return ad.statePaymentText !== 'deny' || ad.statePaymentText !== 'cencel' || ad.statePaymentText !== 'expire' && fraudStatus !== 'deny'
              });
          });
        }

      } catch (error) {
        console.error('Error listing objects:', error);
      }
    };
    getAllDataFromBucket();

  });

  // cancel advertising -y
  socket.on('cancel_ads', async (data) => {
    try {
      socket.emit('att_cnecel_ads')
      const getObjectData = async (key) => {
        const params = {
          Bucket: dbNameAdsWasabi,
          Key: key,
        };
        try {
          const data = await s3.getObject(params).promise();
          return JSON.parse(data.Body.toString());
        } catch (error) {
          console.error(`Error getting object '${key}':`, error);
          throw error;
        }
      };
      const getAllDataFromBucket = async () => {
        const param = {
          Bucket: dbNameAdsWasabi,
        };
        try {
          const listObjectsResponse = await s3.listObjectsV2(param).promise();
          const objects = listObjectsResponse.Contents;
          const allData = await Promise.all(
            objects.map(async (object) => {
              const key = object.Key;
              const data = await getObjectData(key);
              return { key: key, data };
            })
          );
          //remove ad
          const mapAllData = allData.map(dtbs => {
            const dataM = dtbs.data
            return dataM
          })
          const getDltAdData = mapAllData.find(adData => {
            return adData.price.transactionId === data
          });
          // dlt file img
          const paramsFile = {
            Bucket: dbNameFileWasabi,
            Key: getDltAdData.idFile,
          };
          s3.deleteObject(paramsFile, (err, dataY) => {
            if (err) {
              console.error('Error deleting object:', err);
            } else {
              console.log('file ad deleted successfully !');
            }
          });
          // dlt file video
          const paramsFileV = {
            Bucket: dbNameFileWasabi,
            Key: getDltAdData.idFileV,
          };
          s3.deleteObject(paramsFileV, (err, dataY) => {
            if (err) {
              console.error('Error deleting object:', err);
            } else {
              console.log('file ad deleted successfully !');
            }
          });
          // dlt ad
          const params = {
            Bucket: dbNameAdsWasabi,
            Key: getDltAdData.idAd,
          };
          s3.deleteObject(params, (err, dataY) => {
            if (err) {
              console.error('Error deleting object:', err);
            } else {
              console.log('ad deleted successfully !');
            }
          });
        } catch (error) {
          console.error('Error listing objects:', error);
        }
      };
      getAllDataFromBucket();

    } catch (error) {
      console.log(error)
    }
  });

  socket.on('get_location_state_contact', async (data) => {
    try {
      // get data user 
      const params = {
        Bucket: dbNameUsersWasabi,
        Key: data.codeContact,
      };
      s3.getObject(params, (err, dataR) => {
        if (err) {
          console.log(`Data doesn't exist !`, err)
        } else {
          const dataUser = JSON.parse(dataR.Body.toString());
          const contactLocation = dataUser.myLocation;
          const contactStateLocation = dataUser.stateLocation;
          socket.emit('recive_location_contact', { location: contactLocation, stateLocation: contactStateLocation })
        }

      });

    } catch (error) {
      console.log(error)
    }
  })

  // change state ads -y
  socket.on('state_ads_change', async (data) => {
    try {
      const params = {
        Bucket: dbNameUsersWasabi,
        Key: data.codeUserApp,
      };
      s3.getObject(params, (err, dataR) => {
        if (err) {
          console.error('Error getting object:', err);
        } else {
          const existingData = JSON.parse(dataR.Body.toString());
          // Misalnya, kita akan menambahkan properti baru atau memperbarui yang sudah ada
          existingData.stateFirstDoAds = false
          const updatedParams = {
            Bucket: dbNameUsersWasabi,
            Key: data.codeUserApp,
            Body: JSON.stringify(existingData),
            ContentType: 'application/json',
          };

          s3.putObject(updatedParams, (err, dataB) => {
            if (err) {
              console.error('Error updating object:', err);
            } else {
              console.log('successfuly updated data:', dataB);
            }
          });
        }
      });

    } catch (error) {
      console.log(error)
    }
  });

  // -y
  socket.on('sendNotification', async (data) => {
    const { to, title, body } = data;
    // Check if the token is a valid Expo push token
    if (!Expo.isExpoPushToken(to)) {
      console.error(`Invalid Expo push token: ${to}`);
      return;
    }
    // Construct the notification message
    const message = {
      to,
      sound: 'default',
      title,
      body,
    };
    // Send the notification
    try {
      await expo.sendPushNotificationsAsync([message]);
      console.log('Notification sent successfully:', message);
    } catch (error) {
      console.error('Error sending notification:', error);
    }
  });

  // -y  
  socket.on('update_token_device_user', async (data) => {
    // console.log(data);
    try {
      if (data) {
        const params = {
          Bucket: dbNameUsersWasabi,
          Key: data.codeUserApp,
        };
        s3.getObject(params, (err, dataR) => {
          if (err) {
            console.error('Error getting object:', err);
          } else {
            const existingData = JSON.parse(dataR.Body.toString());
            // Misalnya, kita akan menambahkan properti baru atau memperbarui yang sudah ada
            existingData.tokenDevice = data.tokenDevice
            const updatedParams = {
              Bucket: dbNameUsersWasabi,
              Key: data.codeUserApp,
              Body: JSON.stringify(existingData),
              ContentType: 'application/json',
            };

            s3.putObject(updatedParams, (err, dataM) => {
              if (err) {
                console.error('Error updating object:', err);
              } else {
                console.log('successfuly updated my device token !');
              }
            });

          }
        });

        // for group
        // ----  
        console.log('token device has been updated !!');
      }
    } catch (error) {
      console.log(error)
    }
  });

  // -y  get file
  socket.on('get_file', async (data) => {
    try {
      const params = {
        Bucket: dbNameFileWasabi,
        Key: data.idFile
      };

      s3.getObject(params, (err, dataR) => {
        if (err) {
          console.error('Error getting object:', err);
        } else {
          const fileData = dataR.Body.toString()
          socket.emit('recive_file', fileData);
        }
      });
    } catch (error) {
      console.log('err', error)
    }
  });

  socket.on('get_msg', (data) => {
    // console.log(data)
    const params = {
      Bucket: dbNameRoomWasabi,
      Key: data.roomPath
    };

    s3.getObject(params, (err, dataR) => {
      if (err) {
        console.error('Error getting object:', err);
      } else {
        const roomData =JSON.parse( dataR.Body.toString());
        const groupMsg = roomData.message;
        const findMsg = groupMsg.find(msg => {
          return msg.timeSandComplete === data.timeSend
        });
        socket.emit('recive_msg_get', findMsg);
        // socket.emit('recive_file', fileData);
      }
    });
  })

  // remove file
  socket.on('remove_file', async (data) => {
    const params = {
      Bucket: dbNameFileWasabi,
      Key: data.idFile,
    };

    s3.deleteObject(params, (err, dataR) => {
      if (err) {
        console.error('Error deleting object:', err);
      } else {
        console.log('File deleted successfully !');
      }
    });
  });

  // get contact collection user
  socket.on('get_contact_coll', async (data) => {
    try {
      if (data) {
        function getObjectStream(params) {
          return s3.getObject(params).createReadStream();
        }
        const chunks = []; // Simpan bagian-bagian data di sini
        const param = {
          Bucket: dbNameUsersWasabi,
          Key: data.codeUserApp,
        };
        getObjectStream(param)
        .on('data', function (chunk) {
          // Proses setiap bagian data yang diambil
          chunks.push(chunk);
        })
        .on('end',function () {
          const combinedDataC = combineChunks(chunks);
          const dataUser = JSON.parse(combinedDataC.toString());
          const contactColl = dataUser.contactCollections;
          socket.emit('recive_contact_coll', contactColl);
        })
        .on('error', function (err) {
          // Tangani kesalahan jika ada
          console.error('Error:', err);
        });
        // Implementasikan fungsi untuk menggabungkan bagian-bagian data yang telah dipecah
          function combineChunks(chunks) {
            // Gabungkan bagian-bagian data menjadi satu
            return Buffer.concat(chunks);
          }

  
      }
    } catch (error) {
      console.log(error)
    }

  });

  // get group collection user
  socket.on('get_group_coll', async (data) => {
    try {
      if (data) {
        const params = {
          Bucket: dbNameUsersWasabi,
          Key: data.codeUserApp
        };

        s3.getObject(params, async (err, dataR) => {
          if (err) {
            console.log('Message :', err)
          } else {
            const dataUser = JSON.parse(dataR.Body.toString());
            const groupColl = dataUser.groupCollections;
            
            socket.emit('recive_group_coll', groupColl);
          }
        });

      }
    } catch (error) {
      console.log(error);
    }

  });

  // add file to dbs
  socket.on('add_file', (data) => {
    const randomUUIDFIle = uuid.v4()
    const params = {
      Bucket: dbNameFileWasabi,
      Key: randomUUIDFIle,
      Body: JSON.stringify(data), // Mengonversi objek JSON menjadi string JSON
      ContentType: 'application/json', // Tentukan jenis konten sebagai JSON
    };

    s3.putObject(params, (err, dataR) => {
      if (err) {
        console.error('Error adding JSON object:', err);
      } else {
        console.log('file added successfully !',);
        // console.log(params);
      }
    });
  });

  // for try public database file
  socket.on('try', (data) => {
    // Baca file ke dalam buffer
    const uriPath = data.path;
    const params = {
      Bucket: 'bucket_try',
      Key: 'imgM1.jpeg',
      Body: Buffer.from(data.file, 'base64'),
      ACL: 'public-read', // Opsional, untuk memberikan akses publik
    };
    
    s3.upload(params, (err, data) => {
      if (err) {
        console.error('Error uploading file:', err);
      } else {
        console.log('File uploaded successfully:', data.Location);
      }
    });
    

  });

  // detect language
  socket.on('detect_lang', (data) => {
    var text = "Hello! How are you?";
    detectlanguage.detect(data.text).then(function(result) {
      if(result.length != 0) {
        let res = result[0].language
        socket.emit('recive_lang',res) 
      }
        // console.log(JSON.stringify(result[0].language));
    });
  })

  socket.on('disconnect', () => {
    socket.disconnect()
    console.log(` User ${socket.id} disconnected`);
  });

});

server.listen(port, async () => {
  console.log(`Running on Server http://localhost:${port}`);
});


```

### Deploy it in 7 seconds: 

[![Deploy to Cyclic](https://deploy.cyclic.app/button.svg)](https://deploy.cyclic.app/)

