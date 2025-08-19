require('dotenv').config();
const express = require('express');
const { Client, GatewayIntentBits, EmbedBuilder, ActionRowBuilder, StringSelectMenuBuilder, ButtonBuilder, ButtonStyle } = require('discord.js');
const mongoose = require('mongoose');
const schedule = require('node-schedule');
const moment = require('moment-timezone');

// Initialize Express app for Render.com
const app = express();
const PORT = process.env.PORT || 3000;

// Health check endpoint
app.get('/', (req, res) => {
  const timestamp = moment().tz('Asia/Kolkata').format('YYYY-MM-DD HH:mm:ss');
  res.status(200).json({
    status: 'online',
    bot: 'Car Insurance Tracker',
    serverTime: timestamp,
    uptime: process.uptime()
  });
});

// Start Express server
app.listen(PORT, () => {
  const timestamp = moment().tz('Asia/Kolkata').format('YYYY-MM-DD HH:mm:ss');
  console.log(`[${timestamp}] SERVER: Express server running on port ${PORT}`);
});

const client = new Client({
  intents: [
    GatewayIntentBits.Guilds,
    GatewayIntentBits.GuildMessages,
    GatewayIntentBits.GuildMembers,
    GatewayIntentBits.DirectMessages,
    GatewayIntentBits.MessageContent
  ]
});

// Enhanced logging system
class Logger {
  static log(message, type = 'info') {
    const timestamp = moment().tz('Asia/Kolkata').format('YYYY-MM-DD HH:mm:ss');
    const logTypes = {
      info: `\x1b[36m[${timestamp}] INFO: \x1b[0m${message}`,
      success: `\x1b[32m[${timestamp}] SUCCESS: \x1b[0m${message}`,
      warning: `\x1b[33m[${timestamp}] WARNING: \x1b[0m${message}`,
      error: `\x1b[31m[${timestamp}] ERROR: \x1b[0m${message}`,
      debug: `\x1b[35m[${timestamp}] DEBUG: \x1b[0m${message}`
    };
    console.log(logTypes[type] || logTypes.info);
  }

  static database(message, operation) {
    const timestamp = moment().tz('Asia/Kolkata').format('YYYY-MM-DD HH:mm:ss');
    console.log(`\x1b[34m[${timestamp}] DATABASE ${operation ? `(${operation})` : ''}: \x1b[0m${message}`);
  }

  static startup(message) {
    const timestamp = moment().tz('Asia/Kolkata').format('YYYY-MM-DD HH:mm:ss');
    console.log(`\x1b[35m[${timestamp}] STARTUP: \x1b[0m${message}`);
  }
}

// MongoDB Connection
mongoose.connect(process.env.MONGO_URI, {
  useNewUrlParser: true,
  useUnifiedTopology: true
}).then(() => {
  Logger.database('Connected successfully', 'CONNECT');
}).catch(err => {
  Logger.error(`Connection error: ${err.message}`, 'CONNECT');
  process.exit(1);
});

// Database Schema
const carInsuranceSchema = new mongoose.Schema({
  carName: { type: String, required: true },
  numberPlate: { type: String, required: true, unique: true },
  expiryDate: { type: Date, required: true },
  addedBy: { type: String, required: true },
  lastUpdated: { type: Date, default: Date.now }
});
const CarInsurance = mongoose.model('CarInsurance', carInsuranceSchema);

// Helper Functions
function checkRoles(member) {
  const requiredRoles = [
    process.env.ADMIN_ROLE_ID,
    process.env.MANAGER_ROLE_ID,
    process.env.HIGH_COMMAND_ROLE_ID,
    process.env.FOUNDER_ROLE_ID,
    process.env.CO_FOUNDER_ROLE_ID
  ];
  return requiredRoles.some(roleId => member.roles.cache.has(roleId));
}

function getDaysLeft(expiryDate) {
  const now = moment().tz('Asia/Kolkata');
  const expiry = moment(expiryDate).tz('Asia/Kolkata');
  return Math.ceil(expiry.diff(now, 'days', true));
}

function getStatusEmoji(daysLeft) {
  if (daysLeft <= 0) return '❌ EXPIRED';
  if (daysLeft <= 3) return '🚨 URGENT';
  if (daysLeft <= 7) return '⚠️ WARNING';
  return '✅ ACTIVE';
}

// Split array into chunks for pagination
function chunkArray(array, chunkSize) {
  const results = [];
  while (array.length) {
    results.push(array.splice(0, chunkSize));
  }
  return results;
}

// Scheduled Task
const rule = new schedule.RecurrenceRule();
rule.hour = 8;
rule.minute = 30;
rule.tz = 'Asia/Kolkata';

schedule.scheduleJob(rule, async () => {
  try {
    Logger.log('Running scheduled insurance check', 'info');
    const alertChannel = client.channels.cache.get(process.env.ALERT_CHANNEL_ID);
    if (!alertChannel) {
      Logger.error('Alert channel not found', 'SCHEDULED TASK');
      return;
    }

    const cars = await CarInsurance.find();
    const alertList = cars.filter(car => {
      const daysLeft = getDaysLeft(car.expiryDate);
      return daysLeft <= 3;
    });

    if (alertList.length > 0) {
      const carChunks = chunkArray([...alertList], 10); // Split into chunks of 10 cars
      
      for (const [index, chunk] of carChunks.entries()) {
        const embed = new EmbedBuilder()
          .setTitle(index === 0 ? '🚨 INSURANCE EXPIRY ALERT' : `🚨 INSURANCE EXPIRY ALERT (PART ${index + 1})`)
          .setColor(0xFF0000)
          .setDescription(index === 0 ? `**${alertList.length} CAR${alertList.length > 1 ? 'S' : ''} NEED ATTENTION!**` : '')
          .setFooter({ text: `Alert generated at ${moment().tz('Asia/Kolkata').format('DD MMM YYYY hh:mm A')}` })
          .setThumbnail('https://i.imgur.com/7X8CQyG.png');

        chunk.forEach(car => {
          const daysLeft = getDaysLeft(car.expiryDate);
          embed.addFields({
            name: `${getStatusEmoji(daysLeft)} ${car.carName.toUpperCase()} (${car.numberPlate})`,
            value: [
              `📅 **Expiry:** ${moment(car.expiryDate).tz('Asia/Kolkata').format('DD MMM YYYY')}`,
              `⏳ **Days Left:** ${daysLeft}`,
              `👤 **Added By:** ${car.addedBy}`,
              `🔄 **Last Updated:** ${moment(car.lastUpdated).tz('Asia/Kolkata').fromNow()}`
            ].join('\n'),
            inline: false
          });
        });

        const content = index === 0 ? 
          `${[
            `<@&${process.env.FOUNDER_ROLE_ID}>`,
            `<@&${process.env.CO_FOUNDER_ROLE_ID}>`,
            `<@&${process.env.HIGH_COMMAND_ROLE_ID}>`,
            `<@&${process.env.SLAYER_ROLE_ID}>`
          ].join(' ')}\n**IMMEDIATE ATTENTION REQUIRED**` : null;

        await alertChannel.send({
          content: content,
          embeds: [embed]
        });
      }
      Logger.log(`Sent alert for ${alertList.length} cars in ${carChunks.length} parts`, 'success');
    } else {
      Logger.log('No expiring insurances found', 'info');
    }
  } catch (err) {
    Logger.error(`Scheduled task error: ${err.message}`, 'SCHEDULED TASK');
  }
});

// Bot Events
client.on('ready', () => {
  Logger.startup(`Bot logged in as ${client.user.tag}`);
  Logger.startup(`Serving ${client.guilds.cache.size} guild(s)`);
  
  client.application.commands.set(commands, process.env.GUILD_ID)
    .then(() => Logger.log('Slash commands registered', 'success'))
    .catch(err => Logger.error(`Command registration failed: ${err.message}`, 'COMMAND SETUP'));
});

client.on('interactionCreate', async interaction => {
  try {
    if (!interaction.inGuild()) return;
    
    if (interaction.isAutocomplete()) {
      const focusedValue = interaction.options.getFocused();
      const command = interaction.commandName;
      
      if (['add_car_insurance', 'less_car_insurance', 'remove_car_insurance'].includes(command)) {
        try {
          const cars = await CarInsurance.find({
            $or: [
              { carName: { $regex: focusedValue, $options: 'i' } },
              { numberPlate: { $regex: focusedValue, $options: 'i' } }
            ]
          }).limit(25);
          
          const options = cars.map(car => ({
            name: `${car.carName} - ${car.numberPlate} (${getDaysLeft(car.expiryDate)}d)`,
            value: car.numberPlate
          }));
          
          if (options.length === 0 && focusedValue) {
            options.push({
              name: `➕ Add New: "${focusedValue}"`,
              value: 'new_car_' + focusedValue
            });
          }
          
          await interaction.respond(options);
          Logger.database(`Autocomplete for ${command}: ${options.length} options shown`, 'QUERY');
        } catch (err) {
          Logger.error(`Autocomplete error: ${err.message}`, 'AUTOCOMPLETE');
          await interaction.respond([]);
        }
      }
      return;
    }

    if (interaction.isCommand()) {
      Logger.log(`Command received: /${interaction.commandName} from ${interaction.user.tag}`, 'info');
      
      if (!checkRoles(interaction.member)) {
        await interaction.reply({
          content: '⛔ ACCESS DENIED: You lack required permissions',
          ephemeral: true
        });
        Logger.log(`Permission denied for ${interaction.user.tag} on /${interaction.commandName}`, 'warning');
        return;
      }

      switch (interaction.commandName) {
        case 'new_car_insurance':
          await handleNewCarInsurance(interaction);
          break;
        case 'add_car_insurance':
          await handleAddCarInsurance(interaction);
          break;
        case 'less_car_insurance':
          await handleLessCarInsurance(interaction);
          break;
        case 'list_car_insurance':
          await handleListCarInsurance(interaction);
          break;
        case 'alert_insurance':
          await handleAlertInsurance(interaction);
          break;
        case 'dm_car_insurance_list':
          await handleDMCarInsuranceList(interaction);
          break;
        case 'dm_alert_car_insurance':
          await handleDMAlertCarInsurance(interaction);
          break;
        case 'remove_car_insurance':
          await handleRemoveCarInsurance(interaction);
          break;
      }
    }
  } catch (err) {
    Logger.error(`Interaction error: ${err.message}`, 'INTERACTION');
    if (interaction.replied || interaction.deferred) {
      await interaction.followUp({
        content: '❌ SYSTEM ERROR: Command processing failed',
        ephemeral: true
      });
    } else {
      await interaction.reply({
        content: '❌ SYSTEM ERROR: Command processing failed',
        ephemeral: true
      });
    }
  }
});

async function handleNewCarInsurance(interaction) {
  const carName = interaction.options.getString('car_name');
  const numberPlate = interaction.options.getString('number_plate');
  const daysLeft = interaction.options.getInteger('days_left');

  try {
    if (!/^[A-Z0-9-]{2,15}$/i.test(numberPlate)) {
      await interaction.reply({
        embeds: [new EmbedBuilder()
          .setTitle('❌ INVALID NUMBER PLATE')
          .setColor(0xFF0000)
          .setDescription('Please use 2-15 alphanumeric characters (hyphens allowed)')
          .addFields(
            { name: 'Your Input', value: numberPlate, inline: true },
            { name: 'Example', value: 'MH01-AB-1234', inline: true }
          )
        ],
        ephemeral: true
      });
      return;
    }

    const expiryDate = moment().tz('Asia/Kolkata').add(daysLeft, 'days').toDate();
    const newInsurance = new CarInsurance({
      carName,
      numberPlate,
      expiryDate,
      addedBy: interaction.user.tag
    });

    await newInsurance.save();
    Logger.database(`New insurance added: ${carName} (${numberPlate}) by ${interaction.user.tag}`, 'INSERT');
    
    await interaction.reply({
      embeds: [new EmbedBuilder()
        .setTitle('✅ NEW INSURANCE REGISTERED')
        .setColor(0x00FF00)
        .setThumbnail('https://i.imgur.com/JQ6Y5zD.png')
        .addFields(
          { name: '🚗 Car Name', value: carName, inline: true },
          { name: '🔢 Number Plate', value: numberPlate, inline: true },
          { name: '📅 Expiry Date', value: moment(expiryDate).tz('Asia/Kolkata').format('DD MMM YYYY'), inline: true },
          { name: '⏳ Days Valid', value: daysLeft.toString(), inline: true },
          { name: '👤 Registered By', value: interaction.user.tag, inline: true }
        )
        .setFooter({ text: 'Insurance successfully added to database' })
      ]
    });
  } catch (err) {
    if (err.code === 11000) {
      await interaction.reply({
        embeds: [new EmbedBuilder()
          .setTitle('❌ DUPLICATE ENTRY')
          .setColor(0xFF0000)
          .setDescription(`Number plate **${numberPlate}** already exists`)
          .addFields(
            { name: 'Existing Entry', value: 'Check `/list_car_insurance`', inline: true },
            { name: 'Need Help?', value: 'Contact server admins', inline: true }
          )
        ],
        ephemeral: true
      });
      Logger.database(`Duplicate entry attempt: ${numberPlate} by ${interaction.user.tag}`, 'DUPLICATE');
    } else {
      Logger.error(`New car insurance error: ${err.message}`, 'NEW INSURANCE');
      throw err;
    }
  }
}

async function handleAddCarInsurance(interaction) {
  const numberPlate = interaction.options.getString('number_plate');
  const daysToAdd = interaction.options.getInteger('days_to_add');
  
  if (numberPlate.startsWith('new_car_')) {
    const newPlate = numberPlate.replace('new_car_', '');
    await interaction.reply({
      embeds: [new EmbedBuilder()
        .setTitle('🚫 CAR NOT FOUND')
        .setColor(0xFFA500)
        .setDescription(`Use \`/new_car_insurance\` to register:\n**${newPlate}**`)
        .setFooter({ text: 'New vehicles must be registered first' })
      ],
      ephemeral: true
    });
    return;
  }

  try {
    const car = await CarInsurance.findOne({ numberPlate });
    if (!car) {
      await interaction.reply({
        embeds: [new EmbedBuilder()
          .setTitle('❌ RECORD NOT FOUND')
          .setColor(0xFF0000)
          .setDescription(`No insurance found for:\n**${numberPlate}**`)
          .setFooter({ text: 'Check plate number or register new vehicle' })
        ],
        ephemeral: true
      });
      return;
    }

    const oldExpiry = moment(car.expiryDate);
    const newExpiry = oldExpiry.add(daysToAdd, 'days');
    const daysLeft = getDaysLeft(newExpiry);
    
    car.expiryDate = newExpiry.toDate();
    car.lastUpdated = new Date();
    await car.save();
    Logger.database(`Insurance extended: ${car.carName} (${numberPlate}) by ${daysToAdd} days`, 'UPDATE');

    await interaction.reply({
      embeds: [new EmbedBuilder()
        .setTitle('🔄 INSURANCE EXTENDED')
        .setColor(0x00FF00)
        .setThumbnail('https://i.imgur.com/JQ6Y5zD.png')
        .addFields(
          { name: '🚗 Car Name', value: car.carName, inline: true },
          { name: '🔢 Number Plate', value: numberPlate, inline: true },
          { name: '📅 Old Expiry', value: oldExpiry.tz('Asia/Kolkata').format('DD MMM YYYY'), inline: true },
          { name: '📅 New Expiry', value: newExpiry.tz('Asia/Kolkata').format('DD MMM YYYY'), inline: true },
          { name: '⏳ Days Added', value: daysToAdd.toString(), inline: true },
          { name: '⏳ Total Days Left', value: daysLeft.toString(), inline: true },
          { name: '👤 Updated By', value: interaction.user.tag, inline: true }
        )
        .setFooter({ text: `Insurance validity extended successfully` })
      ]
    });
  } catch (err) {
    Logger.error(`Add insurance error: ${err.message}`, 'EXTEND INSURANCE');
    throw err;
  }
}

async function handleLessCarInsurance(interaction) {
  const numberPlate = interaction.options.getString('number_plate');
  const daysToSubtract = interaction.options.getInteger('days_to_subtract');
  
  if (numberPlate.startsWith('new_car_')) {
    const newPlate = numberPlate.replace('new_car_', '');
    await interaction.reply({
      embeds: [new EmbedBuilder()
        .setTitle('🚫 CAR NOT FOUND')
        .setColor(0xFFA500)
        .setDescription(`Use \`/new_car_insurance\` to register:\n**${newPlate}**`)
      ],
      ephemeral: true
    });
    return;
  }

  try {
    const car = await CarInsurance.findOne({ numberPlate });
    if (!car) {
      await interaction.reply({
        embeds: [new EmbedBuilder()
          .setTitle('❌ RECORD NOT FOUND')
          .setColor(0xFF0000)
          .setDescription(`No insurance found for:\n**${numberPlate}**`)
        ],
        ephemeral: true
      });
      return;
    }

    const oldExpiry = moment(car.expiryDate);
    const newExpiry = oldExpiry.clone().subtract(daysToSubtract, 'days');
    const daysLeft = getDaysLeft(newExpiry);
    
    if (daysLeft < 0) {
      await interaction.reply({
        embeds: [new EmbedBuilder()
          .setTitle('❌ INVALID ADJUSTMENT')
          .setColor(0xFF0000)
          .setDescription(`Cannot subtract ${daysToSubtract} days`)
          .addFields(
            { name: 'Current Expiry', value: oldExpiry.tz('Asia/Kolkata').format('DD MMM YYYY'), inline: true },
            { name: 'Resulting Expiry', value: newExpiry.tz('Asia/Kolkata').format('DD MMM YYYY'), inline: true },
            { name: 'Days Left', value: daysLeft.toString(), inline: true }
          )
        ],
        ephemeral: true
      });
      return;
    }

    car.expiryDate = newExpiry.toDate();
    car.lastUpdated = new Date();
    await car.save();
    Logger.database(`Insurance reduced: ${car.carName} (${numberPlate}) by ${daysToSubtract} days`, 'UPDATE');

    await interaction.reply({
      embeds: [new EmbedBuilder()
        .setTitle('⚠️ INSURANCE REDUCED')
        .setColor(0xFFA500)
        .setThumbnail('https://i.imgur.com/7X8CQyG.png')
        .addFields(
          { name: '🚗 Car Name', value: car.carName, inline: true },
          { name: '🔢 Number Plate', value: numberPlate, inline: true },
          { name: '📅 Old Expiry', value: oldExpiry.tz('Asia/Kolkata').format('DD MMM YYYY'), inline: true },
          { name: '📅 New Expiry', value: newExpiry.tz('Asia/Kolkata').format('DD MMM YYYY'), inline: true },
          { name: '⏳ Days Reduced', value: daysToSubtract.toString(), inline: true },
          { name: '⏳ Total Days Left', value: daysLeft.toString(), inline: true },
          { name: '👤 Updated By', value: interaction.user.tag, inline: true }
        )
        .setFooter({ text: 'Insurance validity reduced' })
      ]
    });
  } catch (err) {
    Logger.error(`Less insurance error: ${err.message}`, 'REDUCE INSURANCE');
    throw err;
  }
}

async function handleListCarInsurance(interaction) {
  try {
    const cars = await CarInsurance.find().sort({ expiryDate: 1 });
    
    if (cars.length === 0) {
      await interaction.reply({
        embeds: [new EmbedBuilder()
          .setTitle('📭 NO INSURANCES FOUND')
          .setColor(0x7289DA)
          .setDescription('No car insurances registered yet')
          .setFooter({ text: 'Use /new_car_insurance to add vehicles' })
        ],
        ephemeral: true
      });
      return;
    }

    const carChunks = chunkArray([...cars], 10);
    const embeds = [];

    for (const [index, chunk] of carChunks.entries()) {
      const embed = new EmbedBuilder()
        .setTitle(index === 0 ? '📋 INSURANCE REGISTRY' : `📋 INSURANCE REGISTRY (PART ${index + 1})`)
        .setColor(0x1E90FF)
        .setThumbnail(index === 0 ? 'https://i.imgur.com/JQ6Y5zD.png' : null)
        .setFooter({ 
          text: index === 0 ? `Total Vehicles: ${cars.length} • ${moment().tz('Asia/Kolkata').format('DD MMM YYYY hh:mm A')}` : `Part ${index + 1} of ${carChunks.length}`,
          iconURL: index === 0 ? 'https://i.imgur.com/7X8CQyG.png' : null
        });

      chunk.forEach(car => {
        const daysLeft = getDaysLeft(car.expiryDate);
        embed.addFields({
          name: `${getStatusEmoji(daysLeft)} ${car.carName.toUpperCase()} (${car.numberPlate})`,
          value: [
            `📅 **Expiry:** ${moment(car.expiryDate).tz('Asia/Kolkata').format('DD MMM YYYY')}`,
            `⏳ **Days Left:** ${daysLeft}`,
            `👤 **Added By:** ${car.addedBy}`,
            `🔄 **Last Updated:** ${moment(car.lastUpdated).tz('Asia/Kolkata').fromNow()}`
          ].join('\n'),
          inline: false
        });
      });
      embeds.push(embed);
    }

    if (embeds.length === 1) {
      await interaction.reply({ 
        embeds: embeds,
        ephemeral: true
      });
    } else {
      await interaction.reply({ 
        embeds: [embeds[0]],
        ephemeral: true,
        components: [
          new ActionRowBuilder().addComponents(
            new ButtonBuilder()
              .setCustomId('insurance_list_next')
              .setLabel('Next Page')
              .setStyle(ButtonStyle.Primary)
          )
        ]
      });

      const filter = i => i.user.id === interaction.user.id && i.customId === 'insurance_list_next';
      const collector = interaction.channel.createMessageComponentCollector({ filter, time: 60000 });

      let currentPage = 0;
      collector.on('collect', async i => {
        currentPage++;
        if (currentPage >= embeds.length) currentPage = 0;
        
        await i.update({
          embeds: [embeds[currentPage]],
          components: [
            new ActionRowBuilder().addComponents(
              new ButtonBuilder()
                .setCustomId('insurance_list_prev')
                .setLabel('Previous Page')
                .setStyle(ButtonStyle.Secondary),
              new ButtonBuilder()
                .setCustomId('insurance_list_next')
                .setLabel('Next Page')
                .setStyle(ButtonStyle.Primary)
            )
          ]
        });
      });
    }

    Logger.log(`Insurance list viewed by ${interaction.user.tag} (${cars.length} entries)`, 'info');
  } catch (err) {
    Logger.error(`List insurance error: ${err.message}`, 'LIST INSURANCE');
    throw err;
  }
}

async function handleAlertInsurance(interaction) {
  try {
    const alertChannel = client.channels.cache.get(process.env.ALERT_CHANNEL_ID);
    if (!alertChannel) {
      await interaction.reply({
        content: '❌ Alert channel not configured',
        ephemeral: true
      });
      Logger.error('Alert channel not found in manual alert', 'ALERT');
      return;
    }

    const cars = await CarInsurance.find();
    const expiringCars = cars.filter(car => {
      const daysLeft = getDaysLeft(car.expiryDate);
      return daysLeft <= 3;
    });

    if (expiringCars.length === 0) {
      await interaction.reply({
        embeds: [new EmbedBuilder()
          .setTitle('✅ ALL INSURANCES ACTIVE')
          .setColor(0x00FF00)
          .setDescription('No expiring insurances found')
          .setFooter({ text: 'Next check at 8:30 AM IST' })
        ],
        ephemeral: true
      });
      return;
    }

    const carChunks = chunkArray([...expiringCars], 10);
    
    for (const [index, chunk] of carChunks.entries()) {
      const embed = new EmbedBuilder()
        .setTitle(index === 0 ? '🚨 MANUAL ALERT TRIGGERED' : `🚨 MANUAL ALERT (PART ${index + 1})`)
        .setColor(0xFF0000)
        .setDescription(index === 0 ? `**${expiringCars.length} VEHICLE${expiringCars.length > 1 ? 'S' : ''} NEED ATTENTION!**` : '')
        .setFooter({ 
          text: `Alert generated by ${interaction.user.tag} • ${moment().tz('Asia/Kolkata').format('DD MMM YYYY hh:mm A')}`,
          iconURL: 'https://i.imgur.com/7X8CQyG.png'
        });

      chunk.forEach(car => {
        const daysLeft = getDaysLeft(car.expiryDate);
        embed.addFields({
          name: `${getStatusEmoji(daysLeft)} ${car.carName.toUpperCase()} (${car.numberPlate})`,
          value: [
            `📅 **Expiry:** ${moment(car.expiryDate).tz('Asia/Kolkata').format('DD MMM YYYY')}`,
            `⏳ **Days Left:** ${daysLeft}`,
            `👤 **Added By:** ${car.addedBy}`,
            `🔄 **Last Updated:** ${moment(car.lastUpdated).tz('Asia/Kolkata').fromNow()}`
          ].join('\n'),
          inline: false
        });
      });

      const content = index === 0 ? 
        `${[
          `<@&${process.env.FOUNDER_ROLE_ID}>`,
          `<@&${process.env.CO_FOUNDER_ROLE_ID}>`,
          `<@&${process.env.HIGH_COMMAND_ROLE_ID}>`,
          `<@&${process.env.SLAYER_ROLE_ID}>`
        ].join(' ')}\n**MANUAL ALERT: IMMEDIATE ACTION REQUIRED**` : null;

      await alertChannel.send({
        content: content,
        embeds: [embed]
      });
    }

    await interaction.reply({
      content: `✅ Alert sent to <#${process.env.ALERT_CHANNEL_ID}> (${expiringCars.length} cars in ${carChunks.length} parts)`,
      ephemeral: true
    });
    
    Logger.log(`Manual alert triggered by ${interaction.user.tag} for ${expiringCars.length} cars`, 'info');
  } catch (err) {
    Logger.error(`Alert insurance error: ${err.message}`, 'MANUAL ALERT');
    await interaction.reply({
      content: '❌ Failed to send alert',
      ephemeral: true
    });
  }
}

async function handleDMCarInsuranceList(interaction) {
  try {
    await interaction.deferReply({ ephemeral: true });
    
    // Get all car insurances
    const cars = await CarInsurance.find().sort({ expiryDate: 1 });
    
    if (cars.length === 0) {
      await interaction.editReply({
        embeds: [new EmbedBuilder()
          .setTitle('📭 NO INSURANCES FOUND')
          .setColor(0x7289DA)
          .setDescription('No car insurances to send')
        ],
        ephemeral: true
      });
      return;
    }

    await interaction.editReply({
      embeds: [new EmbedBuilder()
        .setTitle('📤 SEND INSURANCE LIST')
        .setColor(0x7289DA)
        .setDescription('Please mention the user you want to send this to (e.g. @username)')
        .setFooter({ text: 'Reply with a mention within 60 seconds' })
      ],
      ephemeral: true
    });

    // Collect user mention
    const filter = m => m.author.id === interaction.user.id && m.mentions.users.size > 0;
    const collector = interaction.channel.createMessageCollector({ filter, time: 60000 });

    collector.on('collect', async m => {
      try {
        collector.stop();
        const targetUser = m.mentions.users.first();
        
        // Split cars into chunks of 10 for pagination
        const carChunks = chunkArray([...cars], 10);
        
        for (const [index, chunk] of carChunks.entries()) {
          const embed = new EmbedBuilder()
            .setTitle(index === 0 ? '📋 INSURANCE REGISTRY' : `📋 INSURANCE REGISTRY (PART ${index + 1})`)
            .setColor(0x1E90FF)
            .setThumbnail(index === 0 ? 'https://i.imgur.com/JQ6Y5zD.png' : null)
            .setDescription(index === 0 ? `**Current insurance status as of ${moment().tz('Asia/Kolkata').format('DD MMM YYYY hh:mm A')}**` : '')
            .setFooter({ 
              text: index === 0 ? `Total Vehicles: ${cars.length}` : `Part ${index + 1} of ${carChunks.length}`,
              iconURL: index === 0 ? 'https://i.imgur.com/7X8CQyG.png' : null
            });

          chunk.forEach(car => {
            const daysLeft = getDaysLeft(car.expiryDate);
            embed.addFields({
              name: `${getStatusEmoji(daysLeft)} ${car.carName.toUpperCase()} (${car.numberPlate})`,
              value: [
                `📅 **Expiry:** ${moment(car.expiryDate).tz('Asia/Kolkata').format('DD MMM YYYY')}`,
                `⏳ **Days Left:** ${daysLeft}`,
                `👤 **Added By:** ${car.addedBy}`,
                `🔄 **Last Updated:** ${moment(car.lastUpdated).tz('Asia/Kolkata').fromNow()}`
              ].join('\n'),
              inline: false
            });
          });

          try {
            await targetUser.send({ embeds: [embed] });
            if (index === 0) {
              Logger.log(`Insurance list sent to ${targetUser.tag} by ${interaction.user.tag}`, 'info');
            }
          } catch (err) {
            await interaction.editReply({
              embeds: [new EmbedBuilder()
                .setTitle('❌ DM FAILED')
                .setColor(0xFF0000)
                .setDescription(`Could not send DM to ${targetUser}`)
                .setFooter({ text: 'User may have DMs disabled' })
              ],
              ephemeral: true
            });
            Logger.error(`Failed to DM ${targetUser.tag}: ${err.message}`, 'DM');
            return;
          }
        }

        await interaction.editReply({
          embeds: [new EmbedBuilder()
            .setTitle('✅ DM SENT SUCCESSFULLY')
            .setColor(0x00FF00)
            .setDescription(`Insurance list (${cars.length} cars in ${carChunks.length} parts) sent to ${targetUser}`)
          ],
          ephemeral: true
        });
        
        await m.delete().catch(() => {});
      } catch (err) {
        Logger.error(`DM collection error: ${err.message}`, 'DM COLLECT');
      }
    });

    collector.on('end', collected => {
      if (collected.size === 0) {
        interaction.editReply({
          content: '⏲️ You took too long to respond. Command cancelled.',
          ephemeral: true
        });
      }
    });

  } catch (err) {
    Logger.error(`DM insurance error: ${err.message}`, 'DM INSURANCE');
    await interaction.editReply({
      embeds: [new EmbedBuilder()
        .setTitle('❌ ERROR')
        .setColor(0xFF0000)
        .setDescription('An error occurred while processing your request')
      ],
      ephemeral: true
    });
  }
}

async function handleDMAlertCarInsurance(interaction) {
  try {
    await interaction.deferReply({ ephemeral: true });
    
    // Get expiring cars (<= 3 days left)
    const cars = await CarInsurance.find();
    const expiringCars = cars.filter(car => {
      const daysLeft = getDaysLeft(car.expiryDate);
      return daysLeft <= 3;
    });

    if (expiringCars.length === 0) {
      await interaction.editReply({
        embeds: [new EmbedBuilder()
          .setTitle('✅ NO EXPIRING INSURANCES')
          .setColor(0x00FF00)
          .setDescription('No cars need immediate attention')
        ],
        ephemeral: true
      });
      return;
    }

    await interaction.editReply({
      embeds: [new EmbedBuilder()
        .setTitle('📤 SEND ALERT LIST')
        .setColor(0xFFA500)
        .setDescription(`Found ${expiringCars.length} expiring insurances. Mention the user to send alerts to (e.g. @username)`)
        .setFooter({ text: 'Reply with a mention within 60 seconds' })
      ],
      ephemeral: true
    });

    // Collect user mention
    const filter = m => m.author.id === interaction.user.id && m.mentions.users.size > 0;
    const collector = interaction.channel.createMessageCollector({ filter, time: 60000 });

    collector.on('collect', async m => {
      try {
        collector.stop();
        const targetUser = m.mentions.users.first();
        
        // Also send to alert channel if configured
        const alertChannel = process.env.ALERT_CHANNEL_ID ? client.channels.cache.get(process.env.ALERT_CHANNEL_ID) : null;
        
        // Split cars into chunks of 10 for pagination
        const carChunks = chunkArray([...expiringCars], 10);
        
        for (const [index, chunk] of carChunks.entries()) {
          const embed = new EmbedBuilder()
            .setTitle(index === 0 ? '🚨 INSURANCE EXPIRY ALERT' : `🚨 INSURANCE EXPIRY ALERT (PART ${index + 1})`)
            .setColor(0xFF0000)
            .setDescription(index === 0 ? `**${expiringCars.length} CAR${expiringCars.length > 1 ? 'S' : ''} NEED ATTENTION!**` : '')
            .setFooter({ 
              text: `Alert generated by ${interaction.user.tag} • ${moment().tz('Asia/Kolkata').format('DD MMM YYYY hh:mm A')}`,
              iconURL: 'https://i.imgur.com/7X8CQyG.png'
            });

          chunk.forEach(car => {
            const daysLeft = getDaysLeft(car.expiryDate);
            embed.addFields({
              name: `${getStatusEmoji(daysLeft)} ${car.carName.toUpperCase()} (${car.numberPlate})`,
              value: [
                `📅 **Expiry:** ${moment(car.expiryDate).tz('Asia/Kolkata').format('DD MMM YYYY')}`,
                `⏳ **Days Left:** ${daysLeft}`,
                `👤 **Added By:** ${car.addedBy}`,
                `🔄 **Last Updated:** ${moment(car.lastUpdated).tz('Asia/Kolkata').fromNow()}`
              ].join('\n'),
              inline: false
            });
          });

          // Send to mentioned user
          try {
            await targetUser.send({ embeds: [embed] });
            if (index === 0) {
              Logger.log(`Insurance alerts sent to ${targetUser.tag} by ${interaction.user.tag}`, 'info');
            }
          } catch (err) {
            await interaction.editReply({
              embeds: [new EmbedBuilder()
                .setTitle('❌ DM FAILED')
                .setColor(0xFF0000)
                .setDescription(`Could not send DM to ${targetUser}`)
                .setFooter({ text: 'User may have DMs disabled' })
              ],
              ephemeral: true
            });
            Logger.error(`Failed to DM ${targetUser.tag}: ${err.message}`, 'DM ALERT');
            return;
          }

          // Also send to alert channel if configured (only first chunk)
          if (alertChannel && index === 0) {
            const roles = [
              `<@&${process.env.FOUNDER_ROLE_ID}>`,
              `<@&${process.env.CO_FOUNDER_ROLE_ID}>`,
              `<@&${process.env.HIGH_COMMAND_ROLE_ID}>`,
              `<@&${process.env.SLAYER_ROLE_ID}>`
            ].join(' ');

            await alertChannel.send({
              content: `${roles}\n**MANUAL ALERT: IMMEDIATE ACTION REQUIRED**`,
              embeds: [embed]
            });
          }
        }

        await interaction.editReply({
          embeds: [new EmbedBuilder()
            .setTitle('✅ ALERTS SENT SUCCESSFULLY')
            .setColor(0x00FF00)
            .setDescription(`Sent ${expiringCars.length} expiring insurances in ${carChunks.length} parts to ${targetUser}` + 
              (alertChannel ? ` and <#${process.env.ALERT_CHANNEL_ID}>` : ''))
          ],
          ephemeral: true
        });
        
        await m.delete().catch(() => {});
      } catch (err) {
        Logger.error(`DM alert collection error: ${err.message}`, 'DM ALERT COLLECT');
      }
    });

    collector.on('end', collected => {
      if (collected.size === 0) {
        interaction.editReply({
          content: '⏲️ You took too long to respond. Command cancelled.',
          ephemeral: true
        });
      }
    });

  } catch (err) {
    Logger.error(`DM alert insurance error: ${err.message}`, 'DM ALERT INSURANCE');
    await interaction.editReply({
      embeds: [new EmbedBuilder()
        .setTitle('❌ ERROR')
        .setColor(0xFF0000)
        .setDescription('An error occurred while processing your request')
      ],
      ephemeral: true
    });
  }
}

async function handleRemoveCarInsurance(interaction) {
  const numberPlate = interaction.options.getString('number_plate');
  const password = interaction.options.getString('password');
  
  if (numberPlate.startsWith('new_car_')) {
    const newPlate = numberPlate.replace('new_car_', '');
    await interaction.reply({
      embeds: [new EmbedBuilder()
        .setTitle('🚫 INVALID SELECTION')
        .setColor(0xFFA500)
        .setDescription(`Cannot remove unregistered vehicle:\n**${newPlate}**`)
      ],
      ephemeral: true
    });
    return;
  }

  try {
    if (password !== 'slayers1') {
      await interaction.reply({
        embeds: [new EmbedBuilder()
          .setTitle('🔒 ACCESS DENIED')
          .setColor(0xFF0000)
          .setDescription('Incorrect removal password')
          .setFooter({ text: 'Contact leadership for assistance' })
        ],
        ephemeral: true
      });
      Logger.log(`Failed removal attempt by ${interaction.user.tag} (wrong password)`, 'warning');
      return;
    }

    const car = await CarInsurance.findOneAndDelete({ numberPlate });
    if (!car) {
      await interaction.reply({
        embeds: [new EmbedBuilder()
          .setTitle('❌ RECORD NOT FOUND')
          .setColor(0xFF0000)
          .setDescription(`No insurance found for:\n**${numberPlate}**`)
        ],
        ephemeral: true
      });
      return;
    }

    Logger.database(`Insurance removed: ${car.carName} (${numberPlate}) by ${interaction.user.tag}`, 'DELETE');
    await interaction.reply({
      embeds: [new EmbedBuilder()
        .setTitle('🗑️ INSURANCE REMOVED')
        .setColor(0xFF0000)
        .setThumbnail('https://i.imgur.com/7X8CQyG.png')
        .addFields(
          { name: '🚗 Car Name', value: car.carName, inline: true },
          { name: '🔢 Number Plate', value: numberPlate, inline: true },
          { name: '📅 Expiry Date', value: moment(car.expiryDate).tz('Asia/Kolkata').format('DD MMM YYYY'), inline: true },
          { name: '⏳ Days Left', value: getDaysLeft(car.expiryDate).toString(), inline: true },
          { name: '👤 Removed By', value: interaction.user.tag, inline: true }
        )
        .setFooter({ text: 'Permanently deleted from database' })
      ]
    });
  } catch (err) {
    Logger.error(`Remove insurance error: ${err.message}`, 'REMOVE INSURANCE');
    throw err;
  }
}

// Slash Command Definitions
const commands = [
  {
    name: 'new_car_insurance',
    description: 'Register new car insurance',
    options: [
      {
        name: 'car_name',
        description: 'Vehicle make/model (e.g. Toyota Camry)',
        type: 3,
        required: true
      },
      {
        name: 'number_plate',
        description: 'License plate (e.g. MH01-AB-1234)',
        type: 3,
        required: true
      },
      {
        name: 'days_left',
        description: 'Days until insurance expires',
        type: 4,
        required: true,
        min_value: 1
      }
    ]
  },
  {
    name: 'add_car_insurance',
    description: 'Extend existing insurance validity',
    options: [
      {
        name: 'number_plate',
        description: 'Select vehicle from dropdown',
        type: 3,
        required: true,
        autocomplete: true
      },
      {
        name: 'days_to_add',
        description: 'Days to add to insurance',
        type: 4,
        required: true,
        min_value: 1
      }
    ]
  },
  {
    name: 'less_car_insurance',
    description: 'Reduce existing insurance validity',
    options: [
      {
        name: 'number_plate',
        description: 'Select vehicle from dropdown',
        type: 3,
        required: true,
        autocomplete: true
      },
      {
        name: 'days_to_subtract',
        description: 'Days to subtract from insurance',
        type: 4,
        required: true,
        min_value: 1
      }
    ]
  },
  {
    name: 'list_car_insurance',
    description: 'View all registered car insurances (paginated)'
  },
  {
    name: 'alert_insurance',
    description: 'Trigger manual insurance expiry alert (sends to alert channel)'
  },
  {
    name: 'dm_car_insurance_list',
    description: 'Send complete insurance list via DM to mentioned user (paginated)'
  },
  {
    name: 'dm_alert_car_insurance',
    description: 'Send only expiring insurances via DM to mentioned user and alert channel (paginated)',
  },
  {
    name: 'remove_car_insurance',
    description: 'Permanently remove insurance record (password protected)',
    options: [
      {
        name: 'number_plate',
        description: 'Select vehicle to remove',
        type: 3,
        required: true,
        autocomplete: true
      },
      {
        name: 'password',
        description: 'Removal password',
        type: 3,
        required: true
      }
    ]
  }
];

// Start Bot
client.login(process.env.DISCORD_BOT_TOKEN)
  .then(() => Logger.startup('Bot is now running'))
  .catch(err => {
    Logger.error(`Login failed: ${err.message}`, 'STARTUP');
    process.exit(1);
  });

// Process termination handlers
process.on('SIGTERM', () => {
  Logger.log('Received SIGTERM - Shutting down gracefully', 'info');
  client.destroy();
  mongoose.connection.close();
  process.exit(0);
});

process.on('SIGINT', () => {
  Logger.log('Received SIGINT - Shutting down gracefully', 'info');
  client.destroy();
  mongoose.connection.close();
  process.exit(0);
});

process.on('unhandledRejection', (err) => {
  Logger.error(`Unhandled rejection: ${err.message}`, 'UNHANDLED REJECTION');
});

process.on('uncaughtException', (err) => {
  Logger.error(`Uncaught exception: ${err.message}`, 'UNCAUGHT EXCEPTION');
  process.exit(1);
});
